<link rel="stylesheet" href="./css/globals.css">

# choosing a meshing algorithm

You have a populated voxel grid and you need triangles. Maybe you're building a block game and want to render terrain that players can carve at 60 fps. Maybe you're extracting a smooth isosurface from a CT scan. Maybe you're sculpting procedural terrain with caves and overhangs and you need the mesh to stitch seamlessly across different levels of detail. The algorithm you choose will determine the visual character of the result, what your data needs to look like, how much geometry you generate, and how long the process takes.

This page synthesises the four algorithm pages — [blocky and greedy meshing](./blocky-and-greedy-meshing.md), [marching cubes](./marching-cubes.md), [surface nets and dual contouring](./surface-nets-and-dual-contouring.md), and [LOD seams and transvoxel](./lod-seams-and-transvoxel.md) — into a single decision guide. Nothing here replaces those pages; everything here helps you decide which one to read first.

---

## the coarse model — five questions that decide for you

Before reaching for any algorithm, answer five questions about your project. The answers will usually point clearly at one or two candidates.

**What should the result look like?** A blocky, rectilinear aesthetic — every surface axis-aligned — is its own valid visual style and carries the cheapest meshing path. Smooth, curved surfaces require an isosurface algorithm. This is the loudest axis.

**Do you need sharp features?** Flat terrain and organic shapes have no sharp features. Architecture, SDF-sculpted rock with defined ledges, and CAD geometry do. Only dual contouring recovers sharp features automatically; marching cubes and surface nets will round them off.

**Does the mesh need to be watertight or manifold?** Rendering pipelines tolerate a non-manifold mesh. Physics engines, 3D printing, and distance-field pipelines do not. This is a hard requirement, not a preference.

**What does your voxel data actually contain?** Occupancy (a single bit or flag — solid vs. empty) is available in every voxel system. A <em>signed distance field (SDF)</em> — storing at each cell how far that point is from the nearest surface, with sign indicating inside or outside — is richer and enables better surface positioning, but requires you to produce it. Gradient vectors (the direction in which the SDF increases fastest) are needed on top of that for dual contouring's sharp-feature recovery. You cannot use what you didn't store.

**Do your mesh tiles need to stitch across different resolutions?** If you implement LOD chunked terrain, the boundary between a coarse chunk and a fine chunk will crack open unless you apply a seam-filling strategy. Transvoxel is the canonical answer, but it is an add-on to marching cubes (or surface nets), not an algorithm in its own right.

---

## the algorithms and what each is for

### blocky meshing + greedy merging

Take a solid voxel, emit a quad for each of its faces that borders an empty voxel. Cull the hidden ones between solid neighbors. Then, as an optimization pass, <em>greedy meshing</em> merges adjacent coplanar quads of the same type into a single large rectangle, reducing vertex count dramatically — a fully solid 8×8×8 region collapses from 384 quads (after culling) to just 6 quads.

- Requires only **occupancy data** — the simplest possible payload.
- Output is **entirely axis-aligned**. Diagonal surfaces are stairstepped.
- Output is **always watertight and manifold** — each surface is a closed set of quads with no gaps or shared edges between non-adjacent faces.
- Greedy meshing's worst case is a checkerboard pattern (no adjacent same-type faces to merge), where it performs identically to culled meshing. Its best case — large uniform volumes — is near-optimal.
- Meshing cost is proportional to **surface area**, not volume.
- **Does not interact with LOD** in any sophisticated way — you just remesh the coarser chunk independently.

See [blocky and greedy meshing](./blocky-and-greedy-meshing.md) for how the culling and merging passes work.

### marching cubes (+ marching tetrahedra)

For each cube of eight adjacent voxel corners, look up the surface configuration in a precomputed table of 256 cases (reducible to 15 unique patterns by symmetry), and emit the corresponding triangles, positioning each vertex by interpolating along the edge where the scalar field crosses the isovalue. The result is a smooth approximation of the isosurface.

- Requires a **scalar field** — the isovalue can be a density, a distance value, a temperature — anything continuous across the grid. Works with occupancy by treating 0/1 as the scalar.
- Produces **smooth curved surfaces** whose quality improves with grid resolution.
- Has <em>ambiguous cases</em> — configurations where the correct triangulation is geometrically ambiguous — that can produce cracks, holes, or topologically inconsistent geometry in the original 1987 formulation. Later variants (Marching Cubes 33, Chernyaev 1995) extend the lookup table to 33 cases and resolve these, at the cost of implementation complexity [1].
- Sharp features (90° corners, defined edges) are **rounded off** — the algorithm finds the smooth zero crossing and has no mechanism to locate a feature point.
- **Marching tetrahedra** is a variant that decomposes each cube into five or six tetrahedra, each with only 2⁴ = 16 cases. It has no ambiguous cases, always produces an orientable, topologically consistent surface, and was historically preferred when the MC patent (1987–2005) was in force. It generates more triangles per cube than MC for similar quality and is less commonly used today.
- Transvoxel extends marching cubes with transition cells for LOD boundaries. See [LOD seams and transvoxel](./lod-seams-and-transvoxel.md).

See [marching cubes](./marching-cubes.md) for the lookup table structure and the ambiguous-case history.

### surface nets (naive)

Instead of emitting triangles at edge crossings, place one vertex **per grid cell** that contains a surface crossing, at the centroid of that cell (or at a smoothed position derived from the crossing points). Connect adjacent surface-cell vertices with quads. The result is a simpler, more uniform mesh with roughly 40–75% fewer triangles than marching cubes for the same grid [2].

- Requires a **scalar field or SDF** — needs to detect sign changes on grid edges.
- Produces a **smooth surface** similar in quality to marching cubes, without the 256-case lookup table — the implementation is straightforward.
- Can produce **non-manifold vertices** at grid configurations where more than two surface sheets meet at a single cell.
- Sharp features are **not preserved** — vertices stay near cell centers, rounding corners.
- The dual topology (one vertex per cell, one quad per crossed edge) is the key structural difference from marching cubes (vertices on edges, triangles per cell).
- The algorithm is friendly to real-time chunk remeshing because it never needs adjacent-cell data beyond the six face-neighbors to place and connect a vertex.

See [surface nets and dual contouring](./surface-nets-and-dual-contouring.md) for the dual-grid geometry.

### dual contouring

The same dual structure as surface nets — one vertex per surface cell — but vertex position is solved by a <em>quadratic error function (QEF)</em> that minimises the sum of squared distances from the vertex to the surface planes defined by the edge-intersection positions and their gradient normals. When edges converge at a sharp feature, the QEF solution snaps the vertex to that feature point exactly. When the surface is smooth, it approximates it as well as surface nets.

- Requires **Hermite data**: the exact position of each edge-surface crossing **and** the surface normal (SDF gradient) at that crossing. Occupancy alone is not enough; you need a proper SDF with gradients [3].
- <em>Sharp features are reproduced automatically</em> — a right-angle corner or a defined ridge becomes geometrically sharp in the output mesh, not rounded.
- The QEF solve can be **numerically unstable** on nearly-flat surfaces where gradient normals are nearly parallel — the system is under-constrained and the vertex can wander outside the cell, producing self-intersecting geometry. Clamping the vertex to the cell bounds prevents self-intersection but may lose the sharp feature.
- Like surface nets, can produce **non-manifold edges and vertices**. Manifold dual contouring (Schaefer, Ju, Warren 2007) resolves this by decomposing ambiguous cells into tetrahedra, at the cost of additional complexity and potential mesh splitting [4].
- Implementation is substantially more involved than marching cubes: you need gradient data, a linear least-squares solver per cell, and cell-clamping logic.

See [surface nets and dual contouring](./surface-nets-and-dual-contouring.md) for QEF construction and the manifold variant.

### transvoxel (LOD seam filler)

Not a standalone algorithm — an extension of marching cubes for the boundaries between terrain chunks at different LOD levels. A fine chunk and a coarse chunk produce meshes that share a boundary face but with different vertex densities; without intervention, visible cracks appear. Transvoxel inserts <em>transition cells</em> along that boundary: each transition cell operates on nine high-resolution voxel samples from the fine side and three from the coarse side, using a lookup table of 512 cases (73 equivalence classes) to generate crack-free bridging triangles [5].

- Requires the **same scalar/SDF data** as marching cubes — no additional payload.
- Eliminates LOD cracks **completely** when implemented correctly.
- Adds a fixed overhead per chunk boundary face — typically small relative to the chunk interior.
- Applies only when your LOD strategy produces **separate meshes per chunk at different resolutions**. If you use a single mesh for the whole scene (uncommon), it is irrelevant.
- Works with both marching cubes and surface nets (the transition cell concept applies to any grid-edge-based algorithm).

See [LOD seams and transvoxel](./lod-seams-and-transvoxel.md) for transition-cell layout and the lookup table structure.

---

## comparison table

Scores are relative: **high** means the algorithm handles this concern well without special engineering; **low** means it fails there or requires significant work around it.

| | blocky + greedy | marching cubes | marching tetrahedra | surface nets | dual contouring | + transvoxel |
|---|---|---|---|---|---|---|
| **visual style** | blocky, axis-aligned | smooth | smooth | smooth | smooth + sharp | smooth |
| **sharp feature recovery** | native (all edges) | none | none | none | high | none |
| **watertight / manifold** | always | usually (MC33 needed for guarantee) | always | usually (non-manifold vertices possible) | not guaranteed (manifold DC needed) | yes at LOD boundary |
| **data required** | occupancy | scalar field | scalar field | scalar field or SDF | SDF + gradients | same as base |
| **implementation complexity** | low | medium (copy the table) | medium | low–medium | high | medium add-on |
| **triangle count** | low (greedy) | moderate–high | higher than MC | low–moderate | low–moderate | small boundary overhead |
| **LOD seam handling** | trivial (remesh per LOD) | needs transvoxel | needs transvoxel | needs transvoxel | needs equivalent | solves it |
| **SDF/gradient required** | no | no | no | no | yes | no |
| **real-time remesh friendly** | yes | yes | yes | yes | caution (QEF solve) | yes |

---

## use X instead of Y because Z — per use case

### block-building game

<em>Use blocky meshing + greedy merging</em> instead of marching cubes, because player edits land at arbitrary positions in real time and the target aesthetic is rectilinear. Greedy meshing collapses large uniform regions into minimal geometry — a solid 64×64×64 platform becomes six quads, not 24,576. Marching cubes would produce a smooth surface inappropriate to the style, at higher triangle count, for no visual benefit.

The data you need is the simplest possible: a single occupancy bit or material ID per voxel. No SDF, no gradients, no scalar field required.

Add a face-culling pass first (O(n²) surface-area cost instead of O(n³) volume cost), then greedy-merge per chunk in a background thread. The total meshing pipeline can complete in milliseconds for typical chunk sizes (16³ to 32³).

**When to deviate**: if you want ambient occlusion baked into the mesh, or per-face UVs that break across greedy-merged quads, you may find greedy merging harder to exploit. Culled meshing without greedy merging is a valid, simpler fallback that still reduces geometry by 4–6× over naive.

### smooth procedural terrain

<em>Use marching cubes</em> instead of surface nets for most terrain use cases, because marching cubes is widely documented, has freely available lookup tables, and produces predictable results. The smooth surface suits organic cave systems, rolling hills, and overhangs.

Use surface nets instead of marching cubes when triangle count matters and you can afford slightly more complex boundary-handling code. Surface nets produces 40–75% fewer triangles for similar quality, which matters for large streaming worlds where mesh generation is the bottleneck.

Add transvoxel at LOD chunk boundaries in either case — without it, every border between a near chunk and a far chunk will crack open visibly.

Your data should be a signed distance field, not raw occupancy. An SDF gives smoother, better-placed vertices than treating occupancy as a 0/1 scalar field, particularly at coarse resolutions where the grid steps are large.

### medical isosurface

<em>Use marching cubes (with MC33 ambiguity resolution)</em> instead of dual contouring, because medical volumes (CT, MRI) are scalar density fields with no sharp features to recover — the relevant surfaces (bone boundaries, organ walls, tissue transitions) are smooth. Marching cubes was designed for exactly this use case and has decades of validated implementations in VTK and ITK [6].

Dual contouring buys you nothing for smooth anatomy and adds significant implementation burden. The QEF gradient requirement means you would need to compute or store gradient volumes alongside the density data, which increases both storage and compute cost.

The one case where you might choose surface nets instead: multi-label segmentation with many tissue classes, where you need boundaries between adjacent regions (not just between one material and empty). Surface nets handles multi-label boundaries naturally; marching cubes requires separate extraction passes per label pair. The SurfaceNets extension for multi-label data (JCGT 2022) is specifically designed for this [7].

### CAD / SDF sculpting

<em>Use dual contouring</em> instead of marching cubes or surface nets, because the defining property of CAD and SDF-sculpted geometry is sharp, defined features — corners between planes, sharp ridges, and flat faces meeting at a precise angle. Marching cubes rounds every corner; dual contouring reproduces them exactly from the SDF gradients.

The requirement is non-negotiable: you need a proper SDF with gradient vectors at each grid edge crossing. If your sculpting system produces SDFs (as most SDF-based tools do), you have what dual contouring needs.

Use manifold dual contouring (Schaefer, Ju, Warren 2007) [4] rather than the naive variant if the downstream pipeline requires a 2-manifold mesh — for 3D printing, physics, or any operation that assumes the mesh is a closed 2-manifold. The naive variant can produce non-manifold edges at topologically complex configurations.

Be aware: the QEF can produce vertices outside their cells on nearly flat, featureless regions. Clamping to cell bounds prevents self-intersection but may soften features. The tradeoff is inherent to the method.

### LOD planet terrain

<em>Use marching cubes + transvoxel</em> (or surface nets + transvoxel) instead of blocky meshing, because planet-scale terrain needs smooth curved geometry and seamless LOD. At planet scale, chunk boundaries are everywhere — a sphere of chunks has boundaries between every adjacent pair — and every one of them needs stitching.

Transvoxel was designed specifically for this use case and handles the seam between any pair of resolutions (2:1 ratio each step) with a fixed set of transition cells. Planet-specific considerations include:

- The voxel grid must be defined in a coordinate system that tiles spherically — typically cube-sphere projections or icospheres subdivided into voxel chunks. The meshing algorithm itself is coordinate-agnostic, but the chunk hierarchy must be structured appropriately.
- Distant LOD chunks at coarse resolution will have coarser meshes; transvoxel ensures those meshes connect without cracks to finer chunks nearer the viewer.
- Dual contouring can also be paired with a transvoxel-equivalent seam filler if sharp feature recovery matters for the terrain, but the implementation complexity is significantly higher and the visual payoff on organic terrain is modest.

---

## the choice is coupled to the rest of the pipeline

The meshing algorithm does not live in isolation. Two other choices constrain or are constrained by it.

**The data model constrains the algorithm.** If your voxel store holds only occupancy bits, you can use blocky meshing or marching cubes treating occupancy as a scalar — but you cannot use dual contouring, which requires SDF gradients. If your store holds a full signed distance field, all algorithms are available, but the store is larger. See [choosing a voxel store](../storing/choosing-a-voxel-store.md) for how storage layout affects what payloads you can afford.

**The algorithm constrains the render path.** A blocky greedy-meshed output is a static mesh of quads; it goes directly to a rasterizer. A smooth MC or surface nets mesh is a triangle mesh that may need normals computed per-vertex from the SDF gradient or via mesh-normal averaging. A dual-contoured mesh with feature-snapped vertices benefits from explicit sharp normals at feature edges. Volumes that skip meshing entirely (ray-marching the SDF directly) bypass this page completely. See [choosing a render path](../rendering/choosing-a-render-path.md) for how the mesh shape feeds the downstream renderer.

---

## quick reference

| if your project is... | reach for... | avoid... |
|---|---|---|
| block-building / player edits, rectilinear style | blocky + greedy meshing | marching cubes (wrong aesthetic, extra complexity) |
| smooth procedural terrain, no sharp features | marching cubes + transvoxel | dual contouring (gradient cost, no visual gain) |
| smooth terrain, triangle budget tight | surface nets + transvoxel | marching cubes (40–75% more triangles) |
| medical / scientific isosurface, smooth anatomy | marching cubes (MC33) | dual contouring (gradient burden, no sharp features needed) |
| multi-label medical segmentation | surface nets (multi-label variant) | marching cubes (separate pass per label pair) |
| CAD / SDF sculpting, sharp edges required | dual contouring (manifold variant) | marching cubes, surface nets (both round corners) |
| LOD planet terrain, seamless chunks | marching cubes + transvoxel | blocky meshing (wrong aesthetic at terrain scale) |

---

## references

[1] Chernyaev, E. V. (1995). "Marching Cubes 33: Construction of Topologically Correct Isosurfaces." *CERN Technical Report* CN/95-17. [local PDF](../papers/chernyaev-1995-marching-cubes-33.pdf) · [source](https://cds.cern.ch/record/292771)

[2] Newman, T. S. and Yi, H. (2006). "A Survey of the Marching Cubes Algorithm." *Computers & Graphics*, 30(5), 854–879. DOI: 10.1016/j.cag.2006.07.021. [local PDF](../papers/newman-yi-2006-survey-marching-cubes.pdf) · [source](https://cgl.ethz.ch/teaching/scivis_common/Literature/Newman06.pdf)

[3] Ju, T., Losasso, F., Schaefer, S., and Warren, J. (2002). "Dual Contouring of Hermite Data." *ACM Transactions on Graphics* (Proc. SIGGRAPH 2002), 21(3), 339–346. DOI: 10.1145/566654.566586. [local PDF](../papers/ju-losasso-schaefer-warren-2002-dual-contouring-hermite-data.pdf) · [source](https://dl.acm.org/doi/10.1145/566654.566586)

[4] Schaefer, S., Ju, T., and Warren, J. (2007). "Manifold Dual Contouring." *IEEE Transactions on Visualization and Computer Graphics*, 13(3), 610–619. DOI: 10.1109/TVCG.2007.1012. [local PDF](../papers/schaefer-warren-2005-dual-marching-cubes.pdf) · [source](https://www.cs.wustl.edu/~taoju/research/dualsimp_tvcg.pdf)

[5] Lengyel, E. (2010). "Transition Cells for Dynamic Multiresolution Marching Cubes." *Journal of Graphics, GPU, and Game Tools*, 15(2), 1–24. DOI: 10.1080/2151237X.2011.563682. [local PDF](../papers/lengyel-2010-voxel-terrain-transvoxel.pdf) · [source](https://transvoxel.org/)

[6] Gibson, S. F. F. (1998). "Constrained Elastic Surface Nets: Generating Smooth Surfaces from Binary Segmented Data." *Proc. Medical Image Computing and Computer-Assisted Intervention (MICCAI)*, 888–898. (Original surface nets paper for medical segmentation.)

[7] Sellán, S., Ströter, D., Müller, M., Jacobson, A., Stumpp, H., and Sellán, S. (2022). "SurfaceNets for Multi-Label Segmentations with Preservation of Sharp Boundaries." *Journal of Computer Graphics Techniques (JCGT)*, 11(1), 34–54. [source](https://jcgt.org/published/0011/01/03/)
