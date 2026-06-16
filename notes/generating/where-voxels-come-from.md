<link rel="stylesheet" href="./css/globals.css">

# where voxels come from

Storing and rendering voxel data are well-understood problems. Compressing a sparse grid, ray-marching through it, extracting a mesh — all of that machinery is ready to go. But none of it matters until the grid has values in it. Before a single voxel can be stored or rendered, something has to <em>fill the cells</em>. That is what this domain is about.

The four pages that follow each cover one way to fill a grid. This page frames them: why there are four distinct origins, what each one is for, and what kind of payload it tends to produce.

---

## the four origins

Every filled voxel grid comes from one of four sources. They differ in where the data comes from — a mathematical function, an artist's intent, an existing 3D model, or a physical sensor — and that difference shapes everything downstream: what the values mean, how accurate they are, and what you can do with them.

| origin | one-line description | typical payload |
|---|---|---|
| procedural | a function of position returns a value | density or occupancy |
| SDF / CSG modeling | a distance field edited with shape operations | signed distance (SDF) |
| mesh voxelization | an existing triangle mesh stamped onto a grid | occupancy or material |
| scanned / captured | real-world sensors sampled at discrete positions | density (CT/MRI) or SDF (depth fusion) |

The payloads matter. Each downstream stage — surface extraction, rendering, simulation — expects a particular kind of value per cell. Choosing an origin also chooses the payload, which constrains what comes next. [Voxel data models](../foundations/voxel-data-models.md) covers the full payload taxonomy.

---

## procedural generation

You want infinite or arbitrarily large content without storing it all. You write a function that takes a grid coordinate as input and returns a value. Evaluate it at every cell and the grid fills itself.

The workhorse is a <em>noise function</em>: a smooth, pseudo-random scalar field over 3D space. Perlin introduced the gradient noise algorithm that started this tradition in 1985 [1]; later improvements (Simplex noise, domain-warped octaves, ridged multifractal noise) layer complexity at multiple scales to produce terrain with large mountain ranges and small surface detail in a single pass. Caves and overhangs come from a 3D density threshold: if `noise(x, y, z) > threshold`, the cell is solid; otherwise empty. That single rule can produce interconnected cave networks that no heightmap could represent.

**what it produces:** typically a density or occupancy value per cell — "how solid is this point?" SDF output is possible but less common in pure procedural work.

**when to reach for it:**
- terrain, landscapes, cave systems, asteroid fields, anything that needs to be large or infinite
- organic shapes — rock, coral, clouds, smoke — where exact geometry doesn't matter
- rapid prototyping of world structure before artist polish

**what it cannot do:** reproduce a specific shape that exists in the real world, or match geometry an artist has sculpted precisely. For that, you need one of the other three origins.

See [procedural terrain](./procedural-terrain.md) for the algorithms in detail.

---

## SDF and CSG modeling

An artist or tool wants to sculpt a shape directly — adding lumps, carving hollows, smoothing transitions — and have the result live in a voxel grid that supports boolean operations cleanly.

The approach stores, in each cell, <em>the signed distance to the nearest surface</em>: negative inside the shape, positive outside, zero exactly on the surface. A grid of these values is called a signed distance field, or SDF. The surface itself is the implicit set of zero-valued cells — no explicit triangle list needed.

What makes SDFs powerful for modeling is that boolean operations become simple arithmetic. The union of two shapes is the minimum of their two distance values per cell. The intersection is the maximum. The difference (carve shape B from shape A) is the maximum of A and the negation of B. These three operations — union, intersection, difference — are collectively called <em>constructive solid geometry</em> (CSG), and they compose arbitrarily: you build complex shapes by combining simpler primitives [2].

Interactive sculpting tools exploit this: every brush stroke is a local CSG operation on the SDF grid, applied in real time. The result can be re-meshed on the fly with marching cubes, or kept as an SDF for downstream use.

**what it produces:** an SDF grid — each cell holds a float representing signed distance to the surface. This is the most information-rich per-cell payload: it encodes both shape and proximity simultaneously.

**when to reach for it:**
- artist-created or tool-created geometry that needs to support boolean editing
- shapes that will feed surface extraction later (the SDF is already the input marching cubes needs)
- any workflow where you want to blend or smooth between shapes smoothly

See [SDF and CSG modeling](./sdf-and-csg-modeling.md) for how sculpting tools implement this and how distance fields compose.

---

## mesh voxelization

You already have a triangle mesh — exported from a CAD tool, downloaded from a library, generated by a DCC application — and you need it as a voxel grid. Maybe you want to simulate inside it, boolean-combine it with terrain, or feed it to a voxel renderer. Voxelization converts it.

The process is a two-step stamp: first, <em>surface voxelization</em> marks every cell that a triangle overlaps (testing each triangle against the cells in its bounding box, or doing a breadth-first expansion from a seed cell); then <em>solid fill</em> floods the interior, so the result is a solid occupancy grid rather than a hollow shell [3]. A scanline fill algorithm or a ray-parity test determines which cells are inside.

The resolution of the target grid is the critical parameter. A fine mesh voxelized at low resolution will alias badly — thin features disappear, sharp corners round off. Doubling the resolution multiplies cell count by 8, so there is a real cost to going fine. The storage chapter covers how to keep that manageable.

**what it produces:** occupancy (a binary solid/empty flag per cell) is the most common output; material ID grids are also common when the source mesh carries material data. SDF output is possible with a distance-transform pass after voxelization.

**when to reach for it:**
- existing assets (CAD parts, game meshes, scanned triangle meshes) need to enter a voxel pipeline
- you want to boolean-combine a specific model with procedurally generated terrain
- physics or simulation needs a voxel representation of a designed object

The conversion is lossy by nature: the grid cannot represent features smaller than one cell. What you get is an approximation at the chosen resolution, not the original surface.

See [mesh voxelization](./mesh-voxelization.md) for the surface/solid fill algorithms and how to handle thin features and sharp corners.

---

## scanned and captured data

Some voxel grids don't come from a function or an artist — they come directly from the physical world. The sensor samples space at regular positions; the result is already a grid.

Two distinct technologies do this, and they produce different payloads.

### medical and scientific imaging

CT (computed tomography) and MRI (magnetic resonance imaging) scanners reconstruct tissue density from X-ray attenuation or magnetic resonance signals. The output is already a regular 3D grid of scalar values — Hounsfield units for CT, signal intensities for MRI — where each value represents how much matter (and what kind) occupies that point. There is no reconstruction step: the voxel grid *is* the data. The same is true of seismic surveys, electron microscopy, and scientific simulation outputs. These grids typically hold <em>density or intensity values</em>, not binary occupancy.

### lidar and depth camera fusion

Lidar scanners and RGB-D cameras (depth cameras) measure distance to surfaces along rays. Each scan gives a point cloud or depth image — a set of surface samples, not a volume. To turn many such scans into a coherent volumetric representation, the standard approach is <em>depth fusion via a truncated signed distance field</em> (TSDF).

Curless and Levoy introduced the foundational TSDF integration algorithm at SIGGRAPH 1996 [4]: each depth scan is converted to a signed distance function along sensor rays and accumulated into a voxel grid with per-cell weighting. The zero-crossing of the accumulated field is the optimal (least-squares) surface estimate. Truncating the distance function to the vicinity of each surface prevents scans from opposite sides of an object interfering with each other. The result is an SDF grid built incrementally from as many scans as needed — the same algorithm that powers real-time depth fusion in robotics and AR today.

Elfes and colleagues earlier established the probabilistic occupancy grid as a framework for building binary spatial maps from sonar and range sensors in robotics [5], which remains the foundation for lidar-based occupancy mapping.

**what it produces:**
- medical/scientific: density or intensity (continuous float per cell)
- depth fusion: SDF (TSDF grid, ready for marching cubes or direct rendering)

**when to reach for it:**
- you need a digital record of a real physical object or space
- simulation or analysis must match reality, not approximate it
- patient-specific models, as-built architectural surveys, archaeological preservation

See [scanned and volume data](./scanned-and-volume-data.md) for TSDF fusion, sensor noise handling, and how to work with CT/MRI datasets.

---

## origins are not exclusive

Real pipelines chain all four. A game world might use procedural noise to lay down the large-scale terrain and cave structure, voxelize hand-authored mesh props into it, let artists sculpt key regions using SDF/CSG tools, and then simulate debris or erosion on top. The payload at each stage may differ: occupancy from the terrain generator, SDF from the sculpting pass, material IDs from the voxelizer.

This is why the [voxel pipeline](../foundations/the-voxel-pipeline.md) shows generation as a single stage even though it contains four distinct modes: from the downstream stages' perspective, what matters is what comes *out* — a filled grid with a payload type — not how it was filled.

---

## quick reference map

| page | what it covers | payload it produces |
|---|---|---|
| [procedural terrain](./procedural-terrain.md) | noise functions, density thresholds, cave systems | density / occupancy |
| [SDF and CSG modeling](./sdf-and-csg-modeling.md) | distance fields, boolean sculpting, brush operations | SDF |
| [mesh voxelization](./mesh-voxelization.md) | surface voxelization, solid fill, resolution tradeoffs | occupancy / material |
| [scanned and volume data](./scanned-and-volume-data.md) | TSDF fusion, CT/MRI, lidar, depth cameras | density / SDF |

Each page opens with the outcome — what kind of content you are trying to produce — then works down into the algorithm. All four payloads are defined in [voxel data models](../foundations/voxel-data-models.md).

---

## references

[1] Perlin, K. (1985). "An image synthesizer." *ACM SIGGRAPH Computer Graphics*, 19(3), 287–296. DOI: 10.1145/325165.325247. (Introduced gradient noise functions for procedural texture and terrain generation.)

[2] Requicha, A. A. G., and Voelcker, H. B. (1982). "Solid Modeling: A Historical Summary and Contemporary Assessment." *IEEE Computer Graphics and Applications*, 2(2), 9–24. DOI: 10.1109/MCG.1982.1674149. (Foundational reference for constructive solid geometry as a modeling paradigm.)

[3] Aleksandrov, M., Zlatanova, S., and Heslop, D. J. (2021). "Voxelisation Algorithms and Data Structures: A Review." *Sensors*, 21(24), 8241. DOI: 10.3390/s21248241. [source](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8707769/) (Open-access survey of mesh-to-voxel conversion algorithms, surface and solid fill strategies, and voxel data structures.)

[4] Curless, B., and Levoy, M. (1996). "A Volumetric Method for Building Complex Models from Range Images." *Proceedings of SIGGRAPH 1996*, 303–312. DOI: 10.1145/237170.237269. [local PDF](../papers/curless-levoy-1996-tsdf-volumetric-range-images.pdf) · [source](https://graphics.stanford.edu/papers/volrange/volrange.pdf) (Introduced TSDF integration for fusing multiple depth scans into a single SDF voxel grid; foundational for all modern depth fusion pipelines.)

[5] Elfes, A. (1990). "Occupancy Grids: A Stochastic Spatial Representation for Active Robot Perception." *Sixth Conference on Uncertainty in AI*, AAAI. [local PDF](../papers/elfes-1990-occupancy-grids-stochastic-spatial-representation.pdf) · [source](https://arxiv.org/pdf/1304.1098) (Established the probabilistic occupancy grid framework for building binary spatial maps from range sensors.)
