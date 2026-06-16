<link rel="stylesheet" href="./css/globals.css">

# volume ray casting

A radiologist loads a CT scan and rotates a translucent image that shows skin, fat, vessels, and bone all at once — no scalpel, no guesswork about what lies beneath. The same core technique renders the roiling smoke of a film's explosion, letting light scatter through its interior the way it does in the real world. The thing they share: the image is made by looking *through* the volume, not at the surface of it. No surface is extracted. The field itself is the thing being rendered.

That is volume ray casting — the technique of computing what a camera sees by sending a ray through a 3D scalar field and integrating the light that field emits and absorbs along the way.

It is the workhorse of medical imaging workstations, scientific visualization tools, and the VFX pipelines that produce volumetric smoke, fire, and clouds. Understanding it requires three ideas stacked in order: the physical model that says *what light does in a volume*, the computational method that says *how to evaluate it*, and the controls an artist or technician uses to *decide what each density value looks like*.

---

## the physical picture — light inside matter

Before code or equations: what is actually happening when light travels through a cloud, through tissue, through smoke?

Two things happen simultaneously at every point along the path:

1. **Some light is absorbed.** The material soaks up light behind it. The deeper the ray goes, the more of what's behind gets blocked.
2. **Some light is emitted.** The material itself glows — because it's hot, because it's been lit by a surrounding light source and scatters that light toward the camera, or because an artist mapped a color onto that density value.

These two effects together are the <em>emission-absorption optical model</em>. It was first formalized for computer graphics by Kajiya and Von Herzen in 1984 [1] and later refined by Levoy [2] and Max [3] into the standard formulation still in use today.

The intuition is physical and direct: the camera receives light that was emitted somewhere along the ray, *minus* whatever fraction of it was absorbed before it reached the camera. A dense region contributes a lot and blocks a lot. A thin region contributes a little and blocks a little. Empty space contributes and blocks nothing.

---

## the volume rendering integral

The emission-absorption model produces a formal equation — the <em>volume rendering integral</em> — that sums up every contribution along a ray from the near clip plane to the far clip plane.

In plain terms: for each infinitesimally thin slab of material at distance *t* along the ray, compute how much light that slab emits toward the camera, then multiply by how much of that light survives the journey from *t* all the way back to the camera (the transmittance — the fraction not yet absorbed). Add all those contributions together and you have the pixel's color.

The transmittance term is an exponential decay: it starts at 1.0 at the camera and falls toward zero as the ray accumulates absorbing material. A fully opaque region drives transmittance to zero — anything behind it contributes nothing.

This integral cannot be solved analytically for arbitrary volumes, so it is approximated numerically: sample the field at discrete steps along the ray, evaluate the emission and absorption at each step, and sum the contributions.

---

## how it's computed — sampling and compositing

Shooting a ray and accumulating contributions along it requires three mechanical steps.

### sampling the grid

At each step along the ray, the sample point will almost certainly fall between voxels. The value at that position is reconstructed from the eight surrounding grid cells using <em>trilinear interpolation</em> — the 3D generalization of bilinear interpolation on a grid. This produces a smooth scalar value (density, Hounsfield unit, pressure — whatever the field stores) at arbitrary continuous positions, which the rest of the pipeline then processes. See [voxel data models](../foundations/voxel-data-models.md) for how different field representations store the underlying numbers.

### classifying the sample — the transfer function

A raw scalar value from the grid (say, a CT Hounsfield unit) is not a color. A <em>transfer function</em> maps each scalar value to an RGBA tuple — a color and an opacity. Air at −1000 HU might map to fully transparent. Fat might map to a translucent yellow. Cortical bone at +1000 HU might map to bright, nearly opaque white.

The transfer function is the artist's or clinician's primary control over the appearance of the volume. Changing it re-renders the entire image with different material "feels" — bone can be made to disappear while soft tissue is highlighted, or vice versa. The design and editing of transfer functions is explored in depth in [scientific and medical volume rendering](../applications/scientific-and-medical-volume-rendering.md).

Two orders of operation compete here:

| approach | what happens | trade-off |
|---|---|---|
| **post-classification** | interpolate the raw scalar value first, *then* apply the transfer function | preferred — interpolation in data space is well-behaved; gives sharper, higher-quality results |
| **pre-classification** | apply the transfer function to each voxel *before* interpolating | faster to precompute; can produce blurring artifacts near sharp transitions in the transfer function |

Post-classification is the standard for quality rendering; pre-classification is occasionally used for performance when the transfer function is smooth and the difference is acceptable.

### gradient-based shading

A flat composited density image looks like fog — no sense of surface orientation, no three-dimensional structure. To add the perception of shape, the local gradient of the scalar field is estimated at each sample point using central differences between neighboring trilinear samples. That gradient vector points in the direction of maximum density change — perpendicular to the local isosurface — and so it serves as a surface normal.

A Phong illumination model applied to that normal gives the sample a diffuse and specular response to scene lights, making implicit surfaces in the volume appear solid and oriented. This is what makes a CT rendering show the dome of a skull as clearly convex rather than as a uniform haze.

### the over operator — front-to-back compositing

Once each sample has a color and opacity from the transfer function (and optionally shading from the gradient), the samples need to be combined into a single pixel value. The standard method processes samples from the camera outward — <em>front to back</em> — and updates two running accumulators:

- `C` — the accumulated color so far
- `α` — the accumulated opacity so far (starts at 0; approaches 1 as the ray fills up)

At each new sample with color `c_i` and opacity `α_i`, the update is:

```
C  ← C  + (1 − α) × α_i × c_i
α  ← α  + (1 − α) × α_i
```

This is the <em>over operator</em>, the same compositing primitive used in 2D image compositing [3]. The factor `(1 − α)` is the current transmittance — the fraction of the ray that still carries unabsorbed light. Each new sample contributes only as much as the still-open portion of the ray allows. Once `α ≈ 1`, the ray is fully opaque and no further sample can meaningfully change the result.

The alternative, back-to-front compositing, processes samples in reverse order and is algebraically equivalent but loses the transmittance shortcut that makes front-to-back attractive for performance.

---

## performance — the ray doesn't have to go all the way

A naive implementation shoots every ray through the entire bounding box at uniform step size. Two optimizations are standard enough to be considered part of the algorithm.

### early ray termination

Once accumulated opacity exceeds a threshold — typically `α > 0.95` — no further samples are needed: the transmittance is so small that remaining contributions are invisible. The ray is terminated early. This is a direct consequence of using front-to-back compositing with the transmittance factor. In scenes with opaque structures (bone, dense tissue), a large fraction of rays terminate well before the far clip, yielding significant speedup for free.

### empty-space skipping

Volumes are almost always mostly empty. A CT scan of a chest is mostly air. A VFX smoke simulation is mostly vacuum punctuated by density clouds. Stepping through empty voxels wastes time because they contribute nothing to the integral — zero emission, zero absorption.

Empty-space skipping uses a coarse acceleration structure to mark which regions of the volume contain no data above a density threshold. The ray leaps over those regions in one step rather than walking through them sample by sample. The structure can be as simple as a mip-mapped min/max hierarchy or as sophisticated as the hierarchical voxel tree used by VDB.

VDB and NanoVDB are directly relevant here. VDB's tree structure — four levels of spatial nodes encoding occupancy — allows a ray traversal algorithm called HDDA (Hierarchical Digital Differential Analyzer) to skip whole subtrees of empty nodes in a single step [4]. NanoVDB linearizes the VDB tree into a pointer-free, GPU-friendly layout, enabling the same hierarchical skipping on the GPU with no pointer chasing. This is how modern production renderers achieve real-time or near-real-time performance on volumes that would otherwise require billions of steps. The role of VDB in VFX pipelines is covered in [VDB in VFX](../applications/vdb-in-vfx.md).

---

## where this fits in the rendering landscape

Volume ray casting is one of several ways to turn a voxel field into pixels. The comparison matters because different render paths have genuinely different trade-offs.

| | volume ray casting | surface extraction + rasterize |
|---|---|---|
| **what's rendered** | the field itself — translucency, interior structure | a mesh extracted from the field — opaque surfaces |
| **best for** | medical CT/MRI, scientific fields, smoke, fire, clouds | terrain, hard-surface VFX, game objects |
| **artist control** | transfer function (opacity per density) | material on extracted mesh |
| **performance** | expensive — every pixel samples the volume many times | fast on GPU hardware rasterizer |
| **interior visible** | yes — you can see through it | no — interior is hidden behind the surface |

Volume ray casting is the right path when the field *is* the content — when there is no single surface to extract, or when translucency and interior structure are the point. [Ways to render voxels](./ways-to-render-voxels.md) surveys the full set of render paths and when to reach for each. [Choosing a render path](./choosing-a-render-path.md) works through the decision for specific use cases.

The scanned and acquired data that feeds medical and scientific volume renderers is covered in [scanned and volume data](../generating/scanned-and-volume-data.md).

---

## the specifics

### step size and aliasing

The discrete step size along the ray controls both quality and cost. Too large a step misses thin features and produces banding artifacts — the Nyquist limit applies: to reconstruct features at a given scale, you must sample at least twice that frequency. Too small a step burns through sample budget without visible benefit once you're below the voxel spacing. In practice, step sizes between 0.5× and 1.0× the voxel spacing are typical for quality rendering; adaptive schemes that vary step size based on local opacity are used in production systems.

### gradient estimation

Central differences using neighboring trilinear samples are the standard for gradient estimation. The gradient can also be precomputed and stored in a companion volume (a 3-component vector field alongside the scalar field), trading memory for per-sample computation. Precomputed gradients are particularly attractive on the GPU where ALU is cheap and bandwidth is the bottleneck — though storing them doubles or quadruples the memory footprint of the volume.

### 1D vs 2D transfer functions

A 1D transfer function maps a single scalar value to RGBA. A 2D transfer function adds a second axis — typically gradient magnitude — allowing features to be isolated by *both* their density and the sharpness of their boundary. Material boundaries (bone-to-tissue transitions in CT) appear at high gradient magnitude; homogeneous interiors appear at low gradient magnitude. A 2D transfer function can select exactly "the boundary of a specific tissue type" and make it opaque while leaving interior and exterior transparent, producing clean anatomical surface renderings without any mesh extraction.

---

## references

[1] Kajiya, J. T. and Von Herzen, B. P. (1984). "Ray Tracing Volume Densities." *ACM SIGGRAPH Computer Graphics*, 18(3), 165–174. DOI: 10.1145/800031.808594. [source](https://dl.acm.org/doi/10.1145/800031.808594)

[2] Levoy, M. (1988). "Display of Surfaces from Volume Data." *IEEE Computer Graphics and Applications*, 8(3), 29–37. DOI: 10.1109/38.511. [local PDF](../papers/levoy-1988-display-surfaces-volume-data.pdf) · [source](https://graphics.stanford.edu/papers/volume-cga88/volume.pdf)

[3] Max, N. (1995). "Optical Models for Direct Volume Rendering." *IEEE Transactions on Visualization and Computer Graphics*, 1(2), 99–108. DOI: 10.1109/2945.468400. [local PDF](../papers/max-1995-optical-models-direct-volume-rendering.pdf) · [source](https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf)

[4] Museth, K. (2021). "NanoVDB: A GPU-Friendly and Portable VDB Data Structure for Real-Time Rendering and Simulation." *ACM SIGGRAPH Talks*, Article 9. DOI: 10.1145/3450623.3464653.

[5] Drebin, R. A., Carpenter, L., and Hanrahan, P. (1988). "Volume Rendering." *ACM SIGGRAPH Computer Graphics*, 22(4), 65–74. DOI: 10.1145/378456.378484. (Introduced the compositing approach for medical volume visualization; paywalled.)
