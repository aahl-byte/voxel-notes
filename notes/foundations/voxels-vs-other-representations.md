<link rel="stylesheet" href="./css/globals.css">

# voxels vs other representations

You have a 3D thing — a rock, a cityblock, a fluid sim, a scanned skull — and you need to put it in a computer. The first decision you make is also the one that is hardest to undo: how do you *represent* it? That choice determines what operations are cheap, what operations are ruinously expensive, what downstream tools will accept the data, and how much memory you'll need at every resolution. Everything else — meshing, rendering, simulation, machine learning — flows from this one upstream bet.

This page lays out the five serious contenders, describes what each one actually is in plain language, then names it. The goal is a crisp answer to: *when should I reach for a regular volumetric grid instead of one of the others?*

> **Prerequisite:** this page assumes you know what a voxel is as an individual sample. If not, start with [what is a voxel](./what-is-a-voxel.md).

---

## the coarse model — five ways to hold a shape

At the coarsest level, every 3D representation makes the same trade-off: **explicitness vs. implicitness**, and **surface vs. volume**.

- An *explicit* representation directly lists geometry — here are the points, here are the triangles, here are the filled cubes.
- An *implicit* representation stores a *rule* for testing whether a point is inside, on, or outside the shape — you evaluate the rule whenever you need a geometry answer.
- A *surface* representation only stores the shell of an object. The interior is inferred (or absent).
- A *volume* representation stores the whole interior, not just the surface.

That two-by-two gives the rough shape of the field:

|  | surface | volume |
|--|---------|--------|
| **explicit** | triangle mesh, point cloud | voxel grid |
| **implicit** | SDF, neural field | SDF (volumetric), NeRF |

Voxels sit in the explicit-volume corner: they are a uniform grid of samples that fills 3D space, storing a value at every cell. That corner has one big advantage — the interior is always there — and one big cost — memory grows as the *cube* of resolution.

A reader who stops here and remembers only this table holds a true, if coarse, model.

---

## the five representations

### polygon / triangle mesh

A mesh stores a list of vertices (3D positions) and the faces that connect them — most often triangles, because any polygon can be split into triangles and GPUs are optimized for them. The mesh describes only the *surface shell* of an object; the interior is assumed hollow.

**What it is for:** rendering. GPUs have been architecturally optimized for triangle rasterization for three decades. A mesh can capture enormous surface detail at very low memory cost compared to an equivalent voxel grid, because it allocates triangles only where the surface is, not across empty space.

**Where it wins:**
- Render pipelines: every game engine, renderer, and 3D DCC tool speaks triangles natively.
- Detail-per-byte: a single triangle covers a large flat surface; detail concentrates where it is needed.
- Surface properties (normals, tangents, UV maps) are first-class citizens.

**Where it falls down:**
- No interior: a mesh is a shell. If you want to simulate what happens *inside* — fluid flow, voxel destruction, material sampling — the mesh gives you nothing to work with.
- Topology changes are expensive: cutting, merging, or boolean-unioning two meshes requires rewriting connectivity. Non-manifold edges cause failures in most pipelines.
- Boolean edits are fragile: subtracting a sphere from a cube in mesh-space requires robust intersection code that fails on non-manifold input. The same operation in voxel-space or SDF-space is trivial.

**vs. voxels:** choose a mesh when you need fast rendering of a static or near-static surface and don't need to touch the interior. Choose voxels when you need to edit, simulate, or scan-fill the volume.

---

### point cloud

A point cloud is simply an unordered set of 3D positions — (x, y, z) per point, optionally extended with color, intensity, or normals. No connectivity, no triangles, no grid. Points are wherever the scanner happened to fire.

**What it is for:** capture. LiDAR, depth cameras, photogrammetry — all of them produce point clouds directly. The format is the natural output of measurement instruments and requires no reconstruction step to store.

**Where it wins:**
- Trivial to acquire: scan, store, done.
- No preprocessing: you don't need to infer connectivity or fill a grid.
- O(N) memory, where N is the number of points — linear in what you actually measured.
- Machine learning on points is now first-class: Qi et al. (2017) showed in PointNet that you can classify and segment point clouds directly without converting to voxels first, using permutation-invariant pooling [1] ([local PDF](../papers/qi-2017-pointnet-deep-learning-point-sets.pdf), [arxiv](https://arxiv.org/abs/1612.00593)).

**Where it falls down:**
- No connectivity: the cloud doesn't know which points are neighbors on the surface.
- No interior: points sample the *surface*. There is no volumetric fill.
- Noise and varying density: raw scans are messy; operations like normal estimation are fragile.
- Not directly renderable: you have to either splat points (with halos and gaps) or reconstruct a surface first.

**vs. voxels:** use a point cloud when the data came from a scanner and you haven't decided what to do with it yet. Convert to voxels when you need volumetric queries or a regular grid for simulation or learning. Converting the other way — voxels to a point cloud — is trivial (sample the occupied cells).

---

### signed distance field (SDF)

An SDF is a function that, given any point in space, returns a single number: the distance to the nearest surface, positive outside and negative inside. The surface itself is the set of points where the function equals zero — what mathematicians call the zero level set.

You don't store the surface directly. You store a scalar field, and the surface is *implicit* in that field. For practical use, the field is usually sampled onto a grid (giving you a 3D array of floats, much like a voxel grid but storing distance rather than occupancy).

**What it is for:** smooth, topology-agnostic geometry operations. SDFs shine at CSG — boolean union, intersection, and difference reduce to `min`, `max`, and `negate` on the field values, which are uniform grid ops. Topology changes (cutting a hole, merging two blobs) happen automatically because the zero level set just moves; there are no edges to rewire. See [SDF and CSG modeling](../generating/sdf-and-csg-modeling.md) for how this is used in practice.

**Where it wins:**
- Smooth surfaces at any resolution: the continuous function can be sampled as finely as you want.
- CSG is free: boolean operations are arithmetic on scalars.
- Gradient of the SDF is the surface normal: geometric operations like offsetting, rounding, and blending come naturally.
- Sphere tracing: SDFs can be ray-marched efficiently for rendering without ever extracting a mesh.

**Where it falls down:**
- Extraction cost: to get a mesh out of an SDF, you have to run marching cubes (Lorensen & Cline, SIGGRAPH 1987 [2]) or dual contouring (Ju et al., 2002 [3]) across the whole grid. This is a significant pass, especially at high resolution.
- Memory: a dense SDF grid has the same O(n³) cost as a voxel grid — you're still storing one value per cell.
- Pseudo-SDF artifacts: after boolean operations the field may not be a true signed distance anymore; it satisfies the eikonal equation only approximately, which can cause surface reconstruction errors.

**vs. voxels:** SDFs and voxel grids are closely related — the difference is mostly what's *in* each cell (distance vs. occupancy or material value). SDFs are the better choice when you need smooth surfaces and CSG. Voxel occupancy grids are the better choice when you need simple, binary fill for simulation or learning.

---

### neural fields / NeRF

A neural field stores a 3D shape (or scene) as the *weights of a neural network* rather than as explicit geometry. The network is trained to reproduce a scene from images; once trained, you query it by passing any (x, y, z, viewing-direction) coordinate and it returns color and density. The surface is wherever density crosses a threshold — implicit, like an SDF, but encoded in weights rather than a grid.

The canonical example is NeRF (Neural Radiance Field), published by Mildenhall et al. at ECCV 2020 [4] ([arxiv](https://arxiv.org/abs/2003.08934)).

**What it is for:** novel-view synthesis and compact scene capture from photographs. A trained NeRF captures complex appearance effects (reflections, translucency, view-dependent lighting) at a resolution that a voxel grid of comparable perceptual quality would cost far more memory to store. Network weights for a single scene are on the order of 5 MB.

**Where it wins:**
- Photorealistic appearance from images alone.
- Compact representation: the whole scene in megabytes of weights.
- Continuous and resolution-independent: query at any position, no grid aliasing.

**Where it falls down:**
- Slow to train: original NeRF takes hours per scene; faster variants (InstantNGP, 3D Gaussian Splatting) reduce this but add their own limitations.
- Slow to query: each forward pass through the network is expensive; real-time rendering requires baking to an explicit structure.
- Essentially uneditable: the geometry is entangled in the weights. Cutting a hole, changing a material, or extracting a clean mesh is non-trivial.
- Hard to compose: merging two NeRFs into one scene requires retraining or careful hybrid approaches.

**vs. voxels:** choose a neural field when your primary goal is photorealistic view synthesis from photographs and you don't need to edit or simulate the geometry. Choose voxels when you need explicit, editable, composable geometry — or when you need to run simulation over the volume. See [machine learning on voxels](../applications/machine-learning-on-voxels.md) for how neural and voxel approaches are sometimes combined (e.g., voxel feature grids as NeRF accelerators).

---

### voxel grid

A voxel grid is a regular, axis-aligned lattice that fills 3D space, with one value stored at every cell. That value can be anything: occupancy (0/1), a material ID, a density, a temperature, an SDF distance. The grid is the structure; the contents are application-defined.

This is the representation these notes are about. See [what is a voxel](./what-is-a-voxel.md) and [the voxel grid](./the-voxel-grid.md) for the foundation.

**What it is for:** explicit volumetric computation. Any algorithm that needs to ask "what is at position (x, y, z)?" gets a constant-time answer with a single array index. The grid makes no assumptions about what is solid, what is empty, or what the topology looks like — it just holds values.

**Where it wins:**
- <em>Uniform addressing</em>: position maps directly to array index. No pointer chasing, no BVH traversal, no distance evaluation.
- Interiors are explicit: the volume is filled, not just the surface. Simulation, erosion, cavity detection — all straightforward.
- Edits are local and cheap: changing a cell is a single write. Boolean operations are per-cell min/max/mask.
- Regular structure for learning: 3D convolutions work directly on voxel grids without any conversion step.
- Composable and parallelizable: grid cells are independent, making GPU parallelism natural.
- Industry-grade sparse variant: OpenVDB (Museth, SIGGRAPH 2013 [5], [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf)) addresses the memory cost with a hierarchical B+-tree that achieves O(1) average random access while only storing non-empty cells.

**Where it falls down:**
- <em>Memory grows as O(n³)</em>: doubling resolution in each axis costs 8× the memory. A 512³ grid of 4-byte floats is 512 MB. Most real scenes are sparse, so the dense grid wastes most of that on empty space.
- Aliasing and blockiness: the grid imposes a fixed resolution; geometry finer than one cell is lost, and surfaces look stairstepped at low resolution.
- Anisotropy: the grid has three preferred axes. Diagonal features and thin oblique surfaces are harder to represent accurately than axis-aligned ones.
- Not GPU-render-native: GPUs don't rasterize voxels directly; you either raytrace the volume or extract a mesh first. See [why mesh voxels](../meshing/why-mesh-voxels.md).

---

## these representations aren't exclusive — real systems convert

The most important cross-cutting truth: production pipelines don't pick one representation and stay there. They convert between them constantly, choosing whichever form makes the current operation cheap.

Common conversion paths:
- **Mesh → voxels** (voxelization): scan-convert a triangle mesh into a grid to prepare it for simulation or learning. See [mesh voxelization](../generating/mesh-voxelization.md).
- **Voxels → mesh** (isosurface extraction): run marching cubes or dual contouring on a voxel or SDF grid to produce a renderable surface. See [why mesh voxels](../meshing/why-mesh-voxels.md).
- **Point cloud → voxels**: bin points into grid cells to produce an occupancy or density grid.
- **Point cloud → SDF**: fit a Poisson surface reconstruction, then sample the resulting implicit function onto a grid.
- **NeRF → mesh**: march cubes through the density field to extract approximate geometry.
- **Voxels → SDF**: compute distance to the nearest occupied cell for every empty cell.

The practical implication: choosing a representation is choosing your *primary working format*, not declaring that nothing else will ever exist. A VFX pipeline might scan a prop as a point cloud, reconstruct it into an SDF for CSG work, bake the result to voxels for simulation, and finally extract a mesh for rendering — four representations, one prop, each chosen for the operation it enables best.

---

## when to use voxels instead of X — the compact guide

| you want to... | better choice | why not voxels |
|--|--|--|
| render a detailed static surface in real time | **mesh** | meshes render in O(triangles); voxels need meshing or raytracing first |
| store raw scan data from a LiDAR / depth cam | **point cloud** | points are what the sensor produces; voxelizing costs memory before you even know you need it |
| do smooth CSG boolean operations | **SDF** | SDF booleans are single arithmetic ops; voxel booleans lose sub-cell detail |
| synthesize photorealistic novel views from photos | **NeRF / 3DGS** | neural fields capture view-dependent appearance voxels cannot |
| simulate physics across the whole interior | **voxels** | meshes have no interior; SDF/neural fields have no explicit cell state to update |
| edit geometry cell-by-cell (Minecraft-style) | **voxels** | uniform addressing makes local edits O(1) |
| run 3D CNNs on shape for classification/segmentation | **voxels** | regular grid plugs directly into conv layers |
| handle large, mostly-empty outdoor scenes | **sparse voxels (VDB) or SDF** | dense voxels waste O(n³) on air; VDB compresses empty regions |
| composite, boolean, and animate organic shapes at film quality | **SDF + voxels (VDB)** | the industry solution is an SDF stored in a VDB — you get both |

---

## the honest weaknesses of voxels

Even when voxels are the right choice, these costs don't go away:

- **Memory** is the dominant concern. Sparse structures (octrees, VDB) mitigate it, but they add traversal overhead and break the uniform-addressing guarantee.
- **Resolution is a hard ceiling.** Detail finer than one voxel is gone. You can't refine one corner without refining the whole grid (unless you move to an adaptive structure, which adds complexity).
- **Axis-aligned bias.** The grid is square; the world is not. Thin diagonal features, curved surfaces, and fine edges all require higher resolution to represent accurately than a mesh using explicit vertex positions.
- **Not natively renderable.** Every rendering path for voxels — raycasting, raymarching, mesh extraction — adds a step that meshes avoid.

These weaknesses are real but well-understood. The rest of these notes are largely about the algorithms that work around them.

---

## references

[1] Qi, C.R., Su, H., Mo, K., Guibas, L.J. (2017). "PointNet: Deep Learning on Point Sets for 3D Classification and Segmentation." *CVPR 2017*. [arxiv:1612.00593](https://arxiv.org/abs/1612.00593)

[2] Lorensen, W.E., Cline, H.E. (1987). "Marching Cubes: A High Resolution 3D Surface Construction Algorithm." *ACM SIGGRAPH Computer Graphics*, 21(4), 163–169. [ACM DL](https://dl.acm.org/doi/10.1145/37402.37422) *(subscription; widely reproduced in open survey literature)*

[3] Ju, T., Losasso, F., Schaefer, S., Warren, J. (2002). "Dual Contouring of Hermite Data." *ACM SIGGRAPH 2002*.

[4] Mildenhall, B., Srinivasan, P.P., Tancik, M., Barron, J.T., Ramamoorthi, R., Ng, R. (2020). "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis." *ECCV 2020*. [arxiv:2003.08934](https://arxiv.org/abs/2003.08934)

[5] Museth, K. (2013). "VDB: High-Resolution Sparse Volumes with Dynamic Topology." *ACM Transactions on Graphics*, 32(3). [ACM DL](https://dl.acm.org/doi/10.1145/2487228.2487235) — [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf)

[6] Wang, Z. et al. (2024). "3D Representation Methods: A Survey." *arXiv:2410.06475*. [arxiv](https://arxiv.org/abs/2410.06475) — [local PDF](../papers/wang-2024-3d-representation-methods-survey.pdf)
