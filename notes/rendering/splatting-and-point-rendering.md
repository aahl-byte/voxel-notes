<link rel="stylesheet" href="./css/globals.css">

# splatting and point rendering

You have a dataset of millions of points — a laser scan of a building, a particle
simulation, or a learned scene reconstructed from photographs. You need to put it on
screen in real time. Triangulating a mesh from raw point data is fragile: scanners
leave gaps, particles have no surface at all, and learned scene representations
produce points that don't sit on a clean boundary. Tracing rays through the volume
works but is expensive for data this dense. What you want is a third path: take each
point, project it outward onto the screen as a small fuzzy patch, and add all those
patches together. That is the core idea behind <em>splatting</em>, and it is the same
idea powering today's state-of-the-art learned scene renderers.

The [overview of render paths](./ways-to-render-voxels.md) places splatting alongside
ray casting and surface extraction. This page goes deep on how splatting actually
works, when to reach for it, and how it connects to modern 3D Gaussian splatting.

---

## the big picture: going the other way

Every rendering algorithm has to answer one question: for a given pixel on the screen,
what color should it be? The two fundamental strategies differ in which direction they
start from.

**Image-order rendering** asks per pixel: cast a ray from the camera through this
pixel, find what it hits, and compute the color. This is what [volume ray
casting](./volume-ray-casting.md) does. The work is driven by the image — the scene
is passive, answering queries.

**Object-order rendering** asks per object: take this voxel or point, figure out
where it lands on the screen, and deposit its contribution there. The work is driven
by the scene — each element actively pushes its color toward the image.

Splatting is object-order. Instead of pulling color from the scene into pixels, it
pushes color from the scene outward onto the image. Each point or voxel gets
projected to screen space and painted there as a small, soft patch. The final image is
the accumulation of all those patches.

This inversion matters for point clouds. In image-order ray casting, you have to
efficiently find which points intersect each ray — hard when there are millions of
unstructured points and no acceleration structure. In object-order splatting, you
iterate over the points directly, which is exactly what the data already is.

| | ray casting (image-order) | splatting (object-order) |
|---|---|---|
| starting point | each pixel | each scene point / voxel |
| question asked | "what does this pixel see?" | "where does this point land?" |
| scene structure needed | acceleration structure (BVH, octree) | none — just iterate |
| natural fit | solid geometry, surfaces | point clouds, particles, volume samples |
| transparency handling | sample along ray in order | sort-then-composite (see below) |
| parallelism | embarrassingly parallel per pixel | embarrassingly parallel per point |

The detailed tradeoff — including when ray casting beats splatting — is in
[choosing a render path](./choosing-a-render-path.md).

---

## the footprint idea

Here is what each point actually does when it hits the screen.

A voxel or scan point is a sample of a volume at one location. When you project that
point through the camera, it maps to a single pixel coordinate. But a hard point
would produce a single-pixel dot: sharp edges, Moiré artifacts under magnification,
and holes wherever two projected points miss a pixel between them.

Instead, treat each point as the center of a small kernel — a smooth blob of
influence that spreads over a neighborhood of pixels. Project not just the center but
the kernel itself. The kernel's contribution to each pixel it overlaps is determined
by how far that pixel is from the center, weighted by the kernel shape. Add up the
contributions from all the kernels and you have a continuous, smooth image.

That projected kernel — the shape a point leaves on the screen — is its
<em>footprint</em>. Westover (1990) formalized this idea for volume data: each voxel
is a small, soft ball of density, and its footprint on the drawing plane is the
2D projection of that ball [1]. The footprint can be pre-computed and stored in a
lookup table because the kernel shape is separable: for a regular grid of voxels,
all footprints have the same form, just shifted to different pixel locations. This
made splatting fast enough to be practical on 1990s hardware.

### the Gaussian kernel

The most common kernel choice is a Gaussian — a smooth bell curve that falls off with
distance from the center. A few properties make it natural:

- **Smooth:** it has no hard boundary, so adjacent splats blend without visible seams
- **Closed under projection:** when you project a 3D Gaussian through a perspective
  camera, the result is still a Gaussian (just with a different covariance). The math
  stays tractable.
- **Separable:** a 2D Gaussian kernel is the product of two 1D Gaussians, so it can
  be evaluated cheaply by table lookup.
- **Pre-integrable:** the integral of a Gaussian over a pixel footprint has a closed
  form, enabling pre-computation.

A hard disk kernel (flat inside, zero outside) would produce sharp edges between
splats and aliasing wherever the disk boundary clips a pixel. A Gaussian avoids both.

### EWA filtering and antialiasing

A plain Gaussian kernel works well when splats are roughly pixel-sized. Under
minification — when a splat shrinks to sub-pixel size, for instance when you zoom
out — multiple points crowd into a single pixel and aliasing appears. Under
magnification — zooming in — the kernel may not cover enough pixels and holes emerge.

Zwicker et al. (2001) addressed this with <em>EWA volume splatting</em> [2]. EWA
stands for Elliptical Weighted Average, a filter design borrowed from texture
antialiasing. The insight is that the footprint function should combine two things:

1. **A reconstruction kernel** — the Gaussian that represents the underlying sample.
2. **A low-pass prefilter** — a second Gaussian that removes frequencies too high to
   be represented at the current screen resolution.

Combined, these two Gaussians produce a single elliptical Gaussian footprint per
splat. The elliptical shape arises naturally from perspective projection: a spherical
kernel in 3D projects to an ellipse when the surface it represents is tilted relative
to the camera. The projection is computed via the local Jacobian of the
camera-to-screen mapping, and because Gaussians are closed under this transformation,
the combined footprint is still a single Gaussian — just elliptical and oriented to
match the local viewing geometry.

The result is correct antialiasing at all scales: no aliasing under minification,
no holes under magnification, and orientation-aware blending at oblique angles. This
is the mathematical foundation that modern 3D Gaussian splatting inherits directly.

---

## accumulating splats: alpha compositing

Projecting each splat to the screen produces overlapping contributions. You need a
rule for combining them. The standard rule is the <em>over operator</em> from Porter
and Duff's alpha compositing model: each splat has an opacity (alpha) value, and
compositing proceeds as:

```
C_pixel = c₁·α₁ + c₂·α₂·(1−α₁) + c₃·α₃·(1−α₁)·(1−α₂) + ...
```

In words: the first splat (front-most) contributes fully; each subsequent splat is
attenuated by the accumulated opacity of everything in front of it. Once the
accumulated opacity approaches 1, further splats barely contribute — the pixel is
saturated.

This formula requires splats to be composited **in depth order, front to back**. A
splat composited out of order will produce the wrong attenuation, creating visible
artifacts: colors behind opaque surfaces bleeding through, or surfaces that appear
partially transparent when they should be solid.

Sorting is therefore the central cost of splatting when transparency matters. Classic
volume splatting sorted voxels into axis-aligned sheets (perpendicular to the viewing
direction) and composited sheet by sheet — Westover's original algorithm used this
sheet-buffer approach, stepping through sheets back to front [1]. Modern 3D Gaussian
splatting sorts millions of individual Gaussians per frame by their depth value,
using a GPU radix sort on the tile-sorted depth keys.

---

## the through-line to 3D gaussian splatting

The same object-order + footprint-accumulation idea that Westover described in 1990
for volume grids is now the core of the fastest learned-scene renderers.

<em>3D Gaussian splatting</em>, introduced by Kerbl et al. (2023) [3], represents a
photographed scene as a set of millions of anisotropic 3D Gaussians — each one
defined by a 3D position, a full covariance matrix (encoding both size and
orientation), an opacity, and a set of spherical harmonic coefficients for
view-dependent color. These Gaussians are not placed by hand; they are optimized from
a sparse set of calibration images, starting from a point cloud produced by Structure
from Motion.

The method inherits the EWA projection mathematics directly: each 3D Gaussian is
projected to a 2D screen-space footprint by transforming its covariance through the
camera Jacobian — exactly the Zwicker et al. framework. What is new is:

- **Anisotropic covariance:** each Gaussian has an independent shape in 3D
  (a rotation and scale per axis), so it can represent a flat patch, a thin sliver,
  or an elongated blob — whatever the local scene geometry demands. A single Gaussian
  can cover a large flat surface efficiently; a collection of thin ones can represent
  a detailed edge.
- **End-to-end optimization:** position, shape, opacity, and color are all
  differentiable parameters, trained by comparing rendered images to real photographs.
  The renderer is the loss function.
- **Tile-based GPU rasterizer:** the screen is divided into 16×16 pixel tiles.
  Gaussians are sorted and binned per tile, then composited in depth order per tile in
  a single parallel GPU pass. This is what makes real-time rendering possible.
- **Adaptive density control:** during training, Gaussians that are too large or too
  small are split or cloned to improve coverage.

A typical scene uses 200,000 to 2 million Gaussians. Rendering runs at 60–200 FPS at
1080p. Training takes 35–45 minutes [3]. Compare that to NeRF-based volume ray
casting, which requires 10 seconds per frame and 48 hours of training for equivalent
quality — the object-order approach wins on speed by orders of magnitude.

How 3D Gaussian splatting connects to learning voxel-based representations is covered
in [machine learning on voxels](../applications/machine-learning-on-voxels.md).

---

## tradeoffs: what splatting can't do cleanly

Splatting's speed comes with real limitations. Understanding them is part of choosing
the right renderer.

### holes and overlap

If splats are too small relative to their screen-space footprint, gaps appear between
them — the image develops holes wherever no Gaussian center projected nearby. If
splats are too large, they overlap heavily and the image goes blurry; fine detail is
smeared out.

The right splat size depends on the local point density, the viewing distance, and
the camera angle — all of which change per frame. EWA filtering handles this
gracefully for regular data, but for irregular point clouds at varying density it
requires per-point radius estimation or learned parameters. 3DGS sidesteps this by
optimizing the per-Gaussian covariance directly; the training process finds the
size that best explains the photographs.

### depth sorting cost

Correct transparency requires sorting. For a scene with 1 million Gaussians, a GPU
radix sort runs in a few milliseconds — acceptable but not free. The sort must be
repeated every frame as the camera moves. Sorting also assumes Gaussians have a
well-defined depth order, which breaks for extremely large, overlapping Gaussians: a
very large Gaussian's center may be far away while part of it is very close. This
can cause order-dependent artifacts at boundaries between overlapping large splats.

Some recent variants (2024–2025) explore sorting-free stochastic rasterization to
avoid this cost, at some quality cost.

### no hard surfaces

Splats are soft by design. A Gaussian kernel has no hard edge. This makes splatting
natural for smoke, clouds, translucent volumes, and fuzzy learned scenes — but poor
for crisp surface boundaries. A sharp corner rendered as a Gaussian splat always
looks slightly blurred. For geometry where exact silhouettes matter, a surface
extraction approach (marching cubes → mesh) or ray casting against a signed distance
field will produce cleaner results. See [voxels vs other representations](../foundations/voxels-vs-other-representations.md)
for when this matters in practice.

### accumulation accuracy

The over-operator formula is only correct when splats are sorted perfectly. With
approximate sorting (e.g., sorting by Gaussian center depth rather than per-pixel
depth), semi-transparent regions can show visible artifacts. For optically thin
volumes like smoke this is usually acceptable; for hard geometry it is not.

---

## when to reach for splatting

Use splatting when:

- your data is already a point cloud or unstructured sample set (scanner data,
  particles, SfM calibration points)
- you are rendering soft, translucent, or volumetric phenomena (smoke, fire, fog)
- you are working with a learned scene representation (NeRF successor, radiance field)
- real-time or interactive frame rates are required and you can afford the sort
- geometry is not known and meshing would be fragile or impossible

Reach for ray casting ([volume ray casting](./volume-ray-casting.md)) instead when:

- you need physically accurate self-shadowing and scattering through the volume
- the data is on a regular grid and you already have an efficient traversal structure
- hard surface boundaries matter and you need exact occlusion
- you can afford the per-pixel computation cost (offline rendering, medical visualization)

---

## references

[1] Westover, L. (1990). "Footprint evaluation for volume rendering." *Proceedings of
the 17th Annual Conference on Computer Graphics and Interactive Techniques (SIGGRAPH
1990)*, Dallas, TX, pp. 367–376. DOI: 10.1145/97880.97919. (Paywalled — cite by DOI.)

[2] Zwicker, M., Pfister, H., van Baar, J., and Gross, M. (2001). "EWA volume
splatting." *Proceedings of IEEE Visualization 2001*, San Diego, CA.
[local PDF](../papers/zwicker-2001-ewa-volume-splatting.pdf) ·
[source](https://www.cs.umd.edu/~zwicker/publications/EWAVolumeSplatting-VIS01.pdf)

[3] Kerbl, B., Kopanas, G., Leimkühler, T., and Drettakis, G. (2023). "3D Gaussian
splatting for real-time radiance field rendering." *ACM Transactions on Graphics*,
42(4), Article 139. DOI: 10.1145/3592433.
[local PDF](../papers/kerbl-2023-3d-gaussian-splatting.pdf) ·
[source](https://arxiv.org/abs/2308.04079)
