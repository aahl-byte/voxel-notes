<link rel="stylesheet" href="./css/globals.css">

# what is a voxel

Minecraft worlds that players carve apart block by block. CT scans that let surgeons navigate a patient's organs before making a single incision. Smoke and fire in blockbuster films that flows around geometry and lights up naturally. Game terrain that crumbles under an explosion and stays crumbled. 3D-printed parts whose material properties vary cell by cell through the interior.

All of these are built on the same underlying idea: divide a region of 3D space into a regular grid of small cells, and record what is in each one. The cells are the unit of work — the things you read, write, simulate, and render.

That cell is called a <em>voxel</em> — short for *volume element*, the 3D equivalent of a pixel.

---

## the core idea — fill space, don't skin it

The dominant way to describe a 3D object in computer graphics is to trace its surface: store a shell of connected triangles. That shell is accurate and compact when the object is solid and static — but it only describes the boundary. The interior doesn't exist as data.

The voxel approach does something different. Instead of tracing the surface, you take a rectangular region of space and subdivide it uniformly in all three axes — say 256 steps in X, 256 in Y, 256 in Z. Every position on that grid gets a cell. Each cell can hold a value. Fill in the right cells with meaningful values and you have described the shape and content of everything in that region: solid rock, empty air, soft tissue, dense bone.

- **Surface meshes** describe only the skin of an object; the interior is implicit and empty.
- **Voxel grids** describe the contents of a volume; every cell, inside and outside, has a value.

This is why voxels are natural for anything that has an interior that matters — tissue that absorbs radiation differently at different depths, terrain that has underground caves, smoke whose density varies throughout the cloud, materials that need different properties at different points through a solid part.

---

## what a voxel actually is

A voxel is a <em>sample of space at one location on a regular 3D grid</em>. That is the complete definition.

A few things follow from it.

**A voxel is a point, not a box.** The cubic block you see in Minecraft is one way to *render* a voxel — draw a cube centered on the grid point and fill it with a color. But the voxel itself is the sample at that grid location. In medical imaging the same grid is visualized as smooth surfaces or as translucent density fog; in fluid simulation it is never rendered as a cube at all. The cube look is a rendering choice, not the definition.

**A voxel stores whatever the application needs.** The grid structure is fixed and regular; the payload per cell is up to you:

- a single **occupancy bit** — is this cell filled or empty?
- a **density or opacity** — how much matter is here? (CT, MRI, smoke)
- a **material ID** — which of several solid materials fills this cell?
- a **color** — useful for artistic voxel models and scanned point clouds
- a **signed distance** — how far is this cell from the nearest surface, and on which side?
- a **vector field** — velocity or temperature at this point in a fluid simulation

What a voxel "is" in a given system depends entirely on what it stores. The data models behind each of those choices are explored in [voxel data models](./voxel-data-models.md).

**A voxel does not store its own position.** Its location is derived from its index in the grid — row, column, layer — combined with the grid's origin and cell size. This is the same economy that pixels use: a 1920×1080 image does not label each pixel with its (x, y) coordinate; the position is implicit in the array layout.

---

## why this representation at all

Four properties make the voxel grid genuinely useful — not just a curiosity.

### uniform addressing

Given any point in space, converting it to a grid cell is a single arithmetic operation: subtract the grid origin, divide by cell size, floor to an integer. No search, no traversal. Going the other way — from a cell index to the center point in space — is equally direct. This makes voxel grids predictable and fast to index.

### trivial neighbor lookup

The six face-neighbors of a voxel at `(x, y, z)` are at `(x±1, y, z)`, `(x, y±1, z)`, and `(x, y, z±1)`. This is a constant-time offset in an array. Fluid simulation, heat diffusion, cellular automata, and many other algorithms reduce to sweeping over the grid and reading neighbors — and voxel grids make that loop trivially cheap. No pointer chasing, no adjacency lists.

### interiors exist

A voxel grid contains data at every point in the volume, not just on the surface. This makes it natural for:

- **boolean edits** — carve out a sphere by writing "empty" to all cells within that radius
- **simulation** — pressure waves, diffusion, and combustion propagate through the interior, not just along a shell
- **querying** — "is this point inside the object?" is an O(1) lookup, not a ray-cast against a mesh

### data acquired by sampling

CT scanners, MRI machines, seismic sensors, and laser scanners all produce data by sampling space at regular intervals. A voxel grid is the natural container for that data — no conversion step needed, no reconstruction of a surface from points. The grid IS the data.

---

## briefly: why not meshes?

Meshes are the right tool for many jobs — [voxels vs other representations](./voxels-vs-other-representations.md) covers that comparison in full. The short version:

| | triangle mesh | voxel grid |
|---|---|---|
| interior | not stored | exists |
| neighbor lookup | topology traversal | constant-time offset |
| boolean edit | complex CSG | write to cells |
| simulation | surface only | volumetric |
| memory cost | proportional to surface area | proportional to volume |
| smooth curves | exact | approximated at grid resolution |

The last row is where voxels pay a real cost. Smooth, curved geometry is naturally expressed by a mesh; on a voxel grid it is approximated in steps. And the memory cost of a grid scales with the cube of the resolution: doubling the resolution in each axis multiplies memory by 8. A 512³ grid of 4-byte floats needs 512 MB. This is the storage problem that every voxel system eventually has to solve — [the storage problem](../storing/the-storage-problem.md) explores why, and the domain that follows covers the structures people use to keep it manageable.

---

## the specifics

### resolution and the grid

The grid is defined by three numbers — cells in X, cells in Y, cells in Z — and a cell size. Together these fix both the spatial extent of the grid and its resolution (the finest detail it can represent). A cell size of 1 cm means that two features closer than 1 cm apart are indistinguishable. These tradeoffs are examined in [the voxel grid](./the-voxel-grid.md).

### where the word comes from

The word *voxel* was coined by analogy with *pixel* (picture element) and *texel* (texture element): swap the domain prefix, keep the *el* for *element*. The earliest documented use in print is from a 1976 medical imaging proceedings. Arie Kaufman and colleagues at Stony Brook University later formalized the vocabulary of *volume graphics* in a highly cited 1993 survey, establishing the voxel grid as the canonical representation for volumetric data [1].

- **pixel** — pi(cture) + el(ement) — one cell of a 2D image
- **voxel** — vo(lume) + el(ement) — one cell of a 3D volume
- **texel** — tex(ture) + el(ement) — one cell of a texture map

### how a voxel fits into the pipeline

A voxel is stored in a grid, which in turn feeds a rendering or processing pipeline. The full arc — from acquiring or generating voxels, through storing them efficiently, extracting surfaces, and rendering — is the subject of the rest of these notes. For the high-level view of how those stages connect, see [the voxel pipeline](./the-voxel-pipeline.md).

---

## references

[1] Kaufman, A., Cohen, D., and Yagel, R. (1993). "Volume Graphics." *IEEE Computer*, 26(7), 51–64. DOI: 10.1109/MC.1993.274942. (Foundational survey establishing voxels and volume graphics as a discipline.)

[2] Museth, K. (2013). "VDB: High-Resolution Sparse Volumes with Dynamic Topology." *ACM Transactions on Graphics*, 32(3), Article 27. DOI: 10.1145/2487228.2487235. [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf) · [source](https://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf)

[3] Crassin, C., Neyret, F., Sainz, M., Green, S., and Eisemann, E. (2011). "Interactive Indirect Illumination Using Voxel Cone Tracing." *Computer Graphics Forum*, 30(7), 1921–1930. DOI: 10.1111/j.1467-8659.2011.02063.x. [local PDF](../papers/crassin-2011-voxel-cone-tracing.pdf) · [source](https://research.nvidia.com/publication/2011-09_interactive-indirect-illumination-using-voxel-cone-tracing)

[4] Kaufman, A. (1996). "Volume Visualization: Principles and Advances." *ACM SIGGRAPH Course Notes*. [local PDF](../papers/kaufman-1993-volume-visualization-principles-advances.pdf) · [source](https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf)
