<link rel="stylesheet" href="./css/globals.css">

# voxels beyond games

A radiologist scrolls through a CT scan and spots a tumor. A lighting artist in a film studio watches smoke billow around a collapsing building in real time. A renderer in a AAA game engine bounces light off a brick wall so it softly illuminates the ceiling above. A robot arm recognizes a cup on a cluttered table. A dental lab prints a crown whose outer shell is rigid while its core is slightly elastic, all from one print job.

None of these people are thinking about voxels. But every one of those systems runs on the same underlying structure: a regular, addressable, three-dimensional grid where each cell holds a value and every cell's position is implicit in its index.

This domain is about what happens when you take voxels out of game terrain and put them to work in fields where the stakes are higher and the demands are different. Each section below is a separate page — this one maps the territory and explains why a regular grid is specifically the right tool in each case.

---

## the common thread

Before visiting each field, it helps to state the pattern they all share.

A regular 3D grid has three properties that no other representation offers together in one structure:

- **uniform random access** — any cell can be reached in O(1) from its (x, y, z) index, with no tree traversal, no pointer chasing, no search
- **trivial neighbor lookup** — the six face-neighbors of cell (x, y, z) are at (x±1, y, z), (x, y±1, z), (x, y, z±1), which are constant-time array offsets
- **volumetric coverage** — every point in the region has a cell, including the interior; there is no concept of "inside the mesh" vs. "outside the mesh" because the grid simply covers everything

These properties make the same data structure optimal for bouncing light through a scene, reading the density of tissue, simulating smoke, running a convolution, and specifying material at every point in a printed part. The representations differ — the grid might be dense, sparse, or hierarchical, and the payload per cell varies widely — but the addressable regular volume is always what each field is reaching for.

For a deeper comparison with triangle meshes and other representations, see [voxels vs other representations](../foundations/voxels-vs-other-representations.md). For how voxel data moves through a processing system from acquisition to output, see [the voxel pipeline](../foundations/the-voxel-pipeline.md).

---

## the five areas

### real-time global illumination

The goal: a game or interactive scene where surfaces receive not just direct light from lamps and the sky, but also light that has bounced off nearby walls, floors, and objects — the subtle fill light that makes a scene feel real.

Tracing every light path in real time is far too expensive on triangle meshes. The voxel approach, introduced by Crassin et al. in 2011, voxelizes the scene into a coarse grid and stores the direct lighting contribution per cell. To estimate how much bounced light reaches a surface point, it fires a small number of cone-shaped "probes" into the voxel grid, reading the prefiltered radiance stored in a mipmap hierarchy of the grid. Each cone samples a progressively blurred version of the scene as it widens, giving a smooth approximation of the incoming light over a solid angle — in milliseconds per frame, for the full scene.

The regular grid is specifically what makes this possible: the mipmap hierarchy (each level averaging a 2×2×2 block of the level below) maps directly onto the grid structure, so cone sampling at different distances becomes a trilinear lookup into the right mip level. A mesh has no such hierarchy.

This technique, called <em>voxel cone tracing</em>, is the foundation of real-time global illumination in several modern engines [1][2]. The full mechanics are on the [voxel global illumination](./voxel-global-illumination.md) page.

### scientific and medical volume rendering

The goal: visualize the interior of a patient, a rock formation, a fluid simulation, or a materials sample — not just its surface.

CT and MRI scanners work by sampling space on a regular grid. A CT scanner measures X-ray attenuation at each point on a fixed lattice; an MRI measures the nuclear magnetic resonance response at each point. The output is already a regular 3D array of scalar values — one number per cell indicating density, tissue type, or signal intensity. The voxel grid is not a conversion of the data; the data is a voxel grid. Displaying it means reading that grid directly.

Volume rendering algorithms — ray casting, compositing, maximum intensity projection — all walk through the grid along viewing rays, accumulating the values they encounter. Nearest-neighbor lookup, interpolation, and neighbor access are all constant-time array operations. The grid's regularity is not a convenience; it is the reason these algorithms run at interactive rates.

This is <em>scientific and medical volume rendering</em> — the technique of directly visualizing a 3D scalar field without first extracting a surface [3]. A mesh would require a lossy surface reconstruction step before any rendering could happen, and that step discards the interior data that the scan was meant to capture. The dedicated page is [scientific and medical volume rendering](./scientific-and-medical-volume-rendering.md).

### VFX: clouds, smoke, and fire

The goal: simulate and render volumetric phenomena — clouds, fire, smoke, explosions, water spray — at film resolution, where the detail required makes a dense grid completely impractical.

Smoke and fire are not surfaces. They are density fields that vary continuously through a volume and change every frame. A mesh cannot represent them at all; a dense voxel grid would be enormous (a 4096³ grid of floats is 256 GB). The solution is a <em>sparse voxel volume</em>: only the cells that are actually non-empty (above some threshold density) are stored, and they are indexed through a hierarchy of tiles and leaf nodes designed for fast random access.

The dominant format for this in production is <em>VDB</em> (named for its tree structure: VDB = Value + Data B-tree), designed by Ken Museth at DreamWorks and released as OpenVDB in 2012 [4]. VDB stores only the occupied voxels, yet supports the same O(1) random access pattern as a dense grid for any given (x, y, z) coordinate. Film pipelines use VDB volumes as the interchange format between fluid simulation, lighting, and rendering. Every major renderer (RenderMan, Arnold, V-Ray, Karma) reads VDB natively.

The full structure and why it works is on the [VDB in VFX](./vdb-in-vfx.md) page.

### machine learning on 3D data

The goal: let a neural network understand the shape and content of a 3D scene — classify objects, segment regions, detect where things are — the same way a 2D CNN understands an image.

A 2D CNN works by sliding a kernel over a regular grid of pixels. Extending this to 3D requires a regular grid of voxels. When a point cloud or scene scan is voxelized — each point assigned to its enclosing grid cell — the result is a 3D array of occupancy or density values that a <em>3D convolutional neural network</em> can process with exactly the same convolution operation, just extended by one axis [5]. The grid structure is what makes the operation well-defined: convolution requires a regular, addressable neighborhood, and a voxel grid provides that in three dimensions.

The tradeoff against point-based methods like PointNet [6] is direct:

| | voxel grid + 3D CNN | point cloud + PointNet |
|---|---|---|
| convolution | natural, translation-equivariant | requires special architecture |
| memory cost | O(n³) — grows with resolution | O(n) — sparse by nature |
| missing data | robust (empty cell = known value) | sensitive to dropout |
| local structure | captured by kernel implicitly | requires explicit neighbor queries |

Voxel grids win when local geometric context and regular-neighborhood convolutions matter; point clouds win when memory and robustness to missing data are the priority. Hybrid approaches (Point-Voxel CNN) combine both. The [machine learning on voxels](./machine-learning-on-voxels.md) page covers these choices in full.

### digital fabrication

The goal: 3D print a part whose material properties — stiffness, color, conductivity, optical opacity — vary point by point through the interior, not just on the surface.

A surface mesh describes where the boundary of an object is. It says nothing about what the object is made of at each internal location. Multi-material printers (inkjet-style, multinozzle extrusion) operate by depositing material one small droplet or bead at a time on a regular grid, making a decision about which material to use at each position. That decision is naturally expressed as a voxel: the fabrication grid and the data grid are the same structure.

Researchers at MIT and Harvard demonstrated that mapping scientific data sets (CT scans, point cloud measurements, volumetric simulation outputs) directly to printer voxels produces physical objects where the internal material distribution encodes the original data — no intermediate surface reconstruction required [7]. The term for this is <em>voxel printing</em> or, more broadly, <em>digital fabrication at the voxel scale</em>.

The [voxels in fabrication](./voxels-in-fabrication.md) page covers the full workflow, including how slicers translate a voxel model into print instructions.

---

## why each field reaches for the same structure

The five areas look unrelated — games, medicine, film, AI, manufacturing — but they share the same requirement: they need to make decisions about what occupies every point in a bounded region of space, and they need to do it efficiently.

A triangle mesh is the right tool when the surface is the thing and the interior doesn't matter. As soon as the interior matters — because it contains tissue, because light must bounce through it, because a simulation propagates through it, because a convolution must slide over it, because material must be deposited into it — the regular addressable grid is the natural structure. The fields above discovered this independently, and they all landed on the same abstraction.

The comparison to other representations is in [voxels vs other representations](../foundations/voxels-vs-other-representations.md).

---

## where to go next

Each area has its own page in this domain:

- [voxel global illumination](./voxel-global-illumination.md) — voxel cone tracing, mipmap hierarchies, and how modern engines approximate bounced light
- [scientific and medical volume rendering](./scientific-and-medical-volume-rendering.md) — ray casting, transfer functions, and the algorithms that turn a CT scan into a navigable 3D image
- [VDB in VFX](./vdb-in-vfx.md) — the sparse B-tree structure, how simulation pipelines use it, and why it became the production standard
- [machine learning on voxels](./machine-learning-on-voxels.md) — 3D CNNs, the voxelization step, and when to use grids vs. point-based methods
- [voxels in fabrication](./voxels-in-fabrication.md) — multi-material printing, per-voxel material specification, and how the print grid maps to the data grid

---

## references

[1] Crassin, C., Neyret, F., Sainz, M., Green, S., and Eisemann, E. (2011). "Interactive Indirect Illumination Using Voxel Cone Tracing." *Computer Graphics Forum*, 30(7), 1921–1930. DOI: 10.1111/j.1467-8659.2011.02063.x. [local PDF](../papers/crassin-2011-voxel-cone-tracing.pdf) · [source](https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf)

[2] Crassin, C. and Green, S. (2012). "Octree-Based Sparse Voxelization Using The GPU Hardware Rasterizer." In *OpenGL Insights*, CRC Press. [local PDF](../papers/crassin-green-2012-octree-sparse-voxelization-gpu.pdf) · [source](https://www.seas.upenn.edu/~pcozzi/OpenGLInsights/OpenGLInsights-SparseVoxelization.pdf)

[3] Levoy, M. (1988). "Display of Surfaces from Volume Data." *IEEE Computer Graphics and Applications*, 8(3), 29–37. DOI: 10.1109/38.511. [local PDF](../papers/levoy-1988-display-surfaces-volume-data.pdf) · [source](https://graphics.stanford.edu/papers/volvis/volvisA4.pdf)

[4] Museth, K. (2013). "VDB: High-Resolution Sparse Volumes with Dynamic Topology." *ACM Transactions on Graphics*, 32(3), Article 27. DOI: 10.1145/2487228.2487235. [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf) · [source](https://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf)

[5] Maturana, D. and Scherer, S. (2015). "VoxNet: A 3D Convolutional Neural Network for Real-Time Object Recognition." *IEEE/RSJ IROS 2015*, 922–928. [source](https://www.ri.cmu.edu/pub_files/2015/9/voxnet_maturana_scherer_iros15.pdf)

[6] Qi, C. R., Su, H., Mo, K., and Guibas, L. J. (2017). "PointNet: Deep Learning on Point Sets for 3D Classification and Segmentation." *CVPR 2017*. arXiv: 1612.00593. [local PDF](../papers/qi-2017-pointnet-deep-learning-point-sets.pdf) · [source](https://arxiv.org/abs/1612.00593)

[7] Javid, F., Liu, J., Shim, J., Bertoldi, K., et al. (2018). "Making data matter: Voxel printing for the digital fabrication of data across scales and domains." *Science Advances*, 4(5), eaas8652. DOI: 10.1126/sciadv.aas8652. [source](https://www.science.org/doi/10.1126/sciadv.aas8652) (open access)
