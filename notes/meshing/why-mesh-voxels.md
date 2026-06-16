<link rel="stylesheet" href="./css/globals.css">

# why mesh voxels

You have a filled grid — millions of cells each tagged filled or empty, or carrying a density value. You want to draw it on screen. The most direct path to a GPU is a triangle mesh: hand the rasterizer a list of triangles, and it draws them fast with lighting, shadows, and all the hardware you paid for. The problem is that a voxel grid is not a triangle mesh. To use the standard rendering path, you must turn the field into a surface.

That conversion is the job of a <em>meshing algorithm</em>.

This domain is about how that conversion works — and how the shape of your data determines which kind of conversion is even possible.

> There is another path: skip the conversion entirely and ray-march the grid directly in a shader, stepping ray by ray through the volume until you hit something solid. That approach lives in the [rendering domain](../rendering/ways-to-render-voxels.md). It trades GPU triangle hardware for shader compute and is the right choice when you need volumetric effects or never want to deal with mesh updates. This domain is about the mesh path.

---

## the core problem — finding the boundary

A voxel grid contains both filled cells and empty cells. The surface of the object is exactly the boundary between them: every face where a filled cell touches an empty cell is a candidate surface face.

Finding that boundary and turning it into triangles is called <em>iso-surface extraction</em>. "Iso" means "same value" — you pick a threshold (the <em>iso-value</em>) and the surface is the set of all points in the grid where the field crosses that threshold. For a binary occupancy grid the threshold is trivially 0.5 (a cell is either filled or not). For a smooth scalar field like a signed distance field the threshold is usually 0 (the zero level-set, where the field transitions from inside to outside).

The algorithm must find that boundary and emit triangles that approximate it. Everything else in this domain is about how well different algorithms approximate it and what tradeoffs they make.

---

## two families — blocky and smooth

The data payload you store in the grid (covered in [voxel data models](../foundations/voxel-data-models.md)) determines which family of meshing algorithm applies to you. This is the most important fork in the domain.

### occupancy grids → blocky / cubic meshing

When each cell is simply filled or empty — no sub-cell gradient, no distance information — the boundary between filled and empty is a hard step: a cell is either fully inside or fully outside. The natural surface for that boundary is a set of axis-aligned square faces, one wherever a filled cell faces an empty cell.

The output looks cubic. That is not a limitation you can route around with a cleverer algorithm; it is a direct consequence of the data. If the data contains no information about where inside a cell the surface might curve, the best faithful surface you can produce is the cell face.

- **naive face culling** — emit one quad per filled-face-to-empty-face boundary. Simple, fast, always correct.
- **greedy meshing** — merge adjacent coplanar quads of the same material into larger rectangles, slashing vertex count. A solid 8×8×8 cube goes from 384 quads (culled) to 6 quads (greedy). The tradeoff: more complex to implement, slower to generate, and gains shrink as material variety or lighting variation breaks potential merges.

Both produce watertight, consistently-wound meshes — every interior gap is covered by exactly one face. The rest of the blocky meshing story is at [blocky and greedy meshing](./blocky-and-greedy-meshing.md).

### scalar / SDF grids → smooth iso-surface extraction

When each cell stores a continuous scalar value — a density, or a signed distance to the nearest surface — the boundary is not a hard step. The surface curves smoothly through the cells, passing through each filled-to-empty edge at a precise sub-cell location determined by linear interpolation between the cell values at either end of the edge.

This gives the algorithm something it didn't have before: it can place a vertex somewhere *inside* the cell face, not just at the cell corner. The output surface can be smooth.

Two algorithm families handle this case:

- **marching cubes and variants** — place vertices on the *edges* of each cell, one vertex per edge that the iso-surface crosses. Each cell (8 corners, each either inside or outside) produces one of 256 possible triangle configurations. The result is a smooth surface whose quality scales with grid resolution. Details at [marching cubes](./marching-cubes.md).
- **dual contouring and surface nets** — place one vertex *inside* each cell that contains a surface crossing, then connect adjacent cells' vertices to form quads. Dual contouring adds Hermite data (the exact intersection point and surface normal on each crossing edge) to position that interior vertex optimally, which lets it reproduce sharp features — a 90-degree edge or a corner — that marching cubes tends to round off. Details at [surface nets and dual contouring](./surface-nets-and-dual-contouring.md).

---

## what makes a good mesh

Regardless of which algorithm you use, a mesh that came from a voxel grid has to satisfy four properties before the rest of the engine can rely on it.

### watertightness

Every triangle must share its edges with exactly one other triangle, and the entire surface must be closed with no gaps. A mesh with gaps leaks — shadow casting fails, physics collision queries return wrong answers, and shaders that assume a closed surface produce artifacts. Both the blocky and smooth families can guarantee watertightness if implemented carefully; both can also produce holes if edge cases are mishandled (particularly at chunk boundaries, where two independently-meshed regions must stitch together).

### consistent winding and normals

A triangle's normal is determined by the order of its vertices: if you curl the fingers of your right hand from the first vertex to the second to the third, your thumb points in the direction of the normal. All triangles must be wound the same way — outward-facing for solid objects — so the GPU's back-face culling discards the inside faces correctly and lighting calculations point the right direction. An algorithm that processes each cell independently can produce flipped triangles at boundaries if it doesn't enforce a global convention.

### per-cell ambiguity

The <em>ambiguity problem</em> is specific to smooth iso-surface algorithms. In certain cell configurations — when diagonal corners of a cube are inside and the other corners outside — there are two topologically different ways to draw the surface through the cell. Choose differently in adjacent cells and you get a hole. The original Lorensen & Cline (1987) marching cubes [1] used a 15-case lookup table that left some ambiguous configurations unresolved, which is why early marching cubes meshes sometimes had holes. Newman & Yi's 2006 survey [2] catalogues the variants developed to fix this. Dual contouring [3] avoids face ambiguity entirely by placing one vertex per cell rather than per edge.

### vertex count

More triangles means more data to transfer, transform, and rasterize. A naive face-culled blocky mesh of a 64³ mostly-solid chunk produces hundreds of thousands of quads — most of which are invisible interior faces if you forget to cull. Greedy meshing can cut that by an order of magnitude. Smooth meshers produce vertex counts that scale with the amount of surface area, not the volume, but high-resolution grids still produce dense meshes. Vertex count feeds directly into [LOD and chunk strategies](../engines/threading-and-meshing-pipeline.md).

---

## it isn't one-and-done — edits force re-meshing

A static voxel model — a medical scan you're visualizing, a pre-baked terrain — needs to be meshed once. An editable voxel world — a game where players carve and build — must be re-meshed every time something changes.

The standard approach is to divide the world into fixed-size chunks (typically 16³ or 32³ cells). Each chunk has its own mesh. When any cell in a chunk changes, that chunk's mesh is regenerated from scratch. "From scratch" sounds expensive, but the bounded chunk size makes it predictable: regenerating a 32³ chunk's mesh takes a few milliseconds on a background thread.

Two complications arise at chunk edges:

- **boundary culling** — a mesher that only sees its own chunk doesn't know whether the cell just outside its boundary is filled. It will emit a face that should be culled. The fix is to give the mesher a one-cell-wide halo of neighbor data.
- **boundary stitching** — smooth meshers that place vertices based on values from both sides of an edge must be given identical data at the boundary so both chunks' meshes agree on where the surface goes.

Both complications require coordination between chunks. The threading model and update queue that manage this live in [the threading and meshing pipeline](../engines/threading-and-meshing-pipeline.md).

---

## the domain map

The pages in this domain proceed outward-to-inward:

| page | what it covers |
|---|---|
| [blocky and greedy meshing](./blocky-and-greedy-meshing.md) | face culling, greedy merge, binary tricks, when to use each |
| [marching cubes](./marching-cubes.md) | the 256-case lookup, edge interpolation, ambiguity, variants |
| [surface nets and dual contouring](./surface-nets-and-dual-contouring.md) | dual vertex placement, Hermite data, sharp features |
| [choosing a meshing algorithm](./choosing-a-meshing-algorithm.md) | the decision: data type → algorithm family → specific tradeoffs |

The pipeline that takes a finished mesh and hands it to the GPU is the [voxel pipeline](../foundations/the-voxel-pipeline.md). How the mesher fits into a multi-threaded engine — update queues, background workers, LOD swapping — is the [threading and meshing pipeline](../engines/threading-and-meshing-pipeline.md).

---

## references

[1] Lorensen, W. E. and Cline, H. E. (1987). "Marching Cubes: A High Resolution 3D Surface Construction Algorithm." *ACM SIGGRAPH Computer Graphics*, 21(4), 163–169. DOI: 10.1145/37402.37422. [source](https://dl.acm.org/doi/10.1145/37402.37422). (The original algorithm establishing the 256-case lookup table for iso-surface extraction from scalar fields.)

[2] Newman, T. S. and Yi, H. (2006). "A Survey of the Marching Cubes Algorithm." *Computers & Graphics*, 30(5), 854–879. DOI: 10.1016/j.cag.2006.07.021. [local PDF](../papers/newman-yi-2006-survey-marching-cubes.pdf) · [source](https://cgl.ethz.ch/teaching/scivis_common/Literature/Newman06.pdf). (Comprehensive survey of ambiguity problems, variants, and quality properties of marching cubes and its successors.)

[3] Ju, T., Losasso, F., Schaefer, S., and Warren, J. (2002). "Dual Contouring of Hermite Data." *ACM Transactions on Graphics* (Proc. SIGGRAPH 2002), 21(3), 339–346. DOI: 10.1145/566654.566586. [local PDF](../papers/ju-losasso-schaefer-warren-2002-dual-contouring-hermite-data.pdf) · [source](https://www.cs.rice.edu/~jwarren/papers/dualcontour.pdf). (Introduces vertex placement inside cells using Hermite data; eliminates face ambiguity and reproduces sharp features.)
