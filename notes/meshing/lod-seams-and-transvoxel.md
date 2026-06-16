<link rel="stylesheet" href="./css/globals.css">

# lod seams and transvoxel

Imagine a planet-scale voxel world — caves, overhangs, cliffs — rendered in real time. The terrain close to the camera is meshed at full resolution: every voxel cell contributes fine detail. Terrain farther away is meshed at half or quarter resolution to save GPU cost. Both approaches use [marching cubes](./marching-cubes.md) or a related algorithm, and they produce clean meshes individually.

The problem is what happens at the boundary where a fine-resolution chunk meets a coarse-resolution chunk. The iso-surfaces on each side were computed independently, at different grid spacings, and their edges do not meet. The seam tears open into visible gaps — sky showing through solid rock, flickering slivers that shadow incorrectly — and no amount of tweaking the per-chunk generation fixes it, because the mismatch is structural.

That gap-at-the-boundary is the subject of this page. Fixing it cleanly requires either hiding the problem or rethinking what "a cell at the boundary" actually means. The principled fix is the <em>Transvoxel algorithm</em>, which inserts special geometry — called <em>transition cells</em> — that bridge the two resolutions in a watertight way.

---

## why gaps form

Large voxel terrain is split into chunks. Each chunk is a fixed-size block of voxel data (Lengyel's implementation uses 16×16×16 cells per block [1]). A LOD system assigns a resolution to each block: blocks near the camera use full resolution, blocks farther away use half resolution (the same physical volume but sampled at half the density), quarter resolution farther still, and so on. Each lower level contains exactly eight blocks of the next finer level — a natural octree.

When marching cubes runs over a full-resolution block, it places iso-surface vertices at sub-cell positions determined by the voxel values at that resolution's sample points. When it runs over a half-resolution block, it places vertices at positions determined by coarser samples. Even if the two blocks share a face in world space, the vertices along that shared face do not agree: they came from different sample densities and are interpolated between different pairs of corners. The edges on either side of the seam are therefore not coincident, and the mesh is not closed.

This kind of gap — where a vertex from the fine mesh sits in the interior of a triangle edge from the coarse mesh, or vice versa — is called a <em>T-junction</em>. T-junctions leave visible cracks, cause z-fighting and shadow errors, and break any algorithm that assumes watertight geometry.

```
Diagram (top view of a 2D seam):

  fine side  |  coarse side
  ---------- | ----------
  ·  ·  ·  · | ·           ·
  v1       v2 | V1             V2
              |
  (fine mesh places 2 verts along the face)
  (coarse mesh places 0 — or 1 at a different position)
  => gap between v2 and V1
```

---

## the cheap fixes and what they get wrong

Two approaches are commonly used before reaching for a proper solution.

### skirts

A skirt is a thin vertical strip of geometry added to the bottom (or whichever face is the seam face) of the fine-resolution mesh. It drops down far enough to overlap the coarse mesh and hide the crack from view. Skirts are easy to compute — each boundary edge of the fine mesh gets a quad dropped from it, and no knowledge of the neighboring chunk is needed. This makes them cache-friendly and trivially parallel.

The problems:

- Skirts hide gaps from most angles but not all. If the terrain is steep or nearly tangent to the seam plane, the skirt may be too short, or it may poke through the coarse mesh surface.
- Skirts cause **z-fighting**: the skirt polygon and the coarse mesh polygon occupy nearly the same depth, and the GPU alternates between them pixel by pixel. Depth-offset hacks can partially fix this but break transparent materials (water, glass) entirely [2].
- Skirts leave the mesh topologically broken. The two sides are still not connected, so shadow maps, ambient occlusion, and watertight intersection tests all produce incorrect results.

### manual stitching

For height-field terrain (where the surface is always a function of x and y), it is straightforward to stitch two adjacent mesh edges by inserting explicit triangles between them. The mesh is always upward-facing, so the stitching geometry is simple to generate.

Voxel terrain breaks this assumption. The surface can be vertical, inverted, or nearly parallel to the seam face. A giant hole can appear where a low-resolution surface is nearly tangent to the boundary plane — a topology that does not exist in height-field terrain at all [1]. Manual stitching strategies developed for height-field terrain fail on these cases and cannot be fixed incrementally.

| approach | watertight | transparent-safe | handles overhangs |
|---|---|---|---|
| skirts | no | no | barely |
| manual stitching | partial | yes | no |
| transition cells | yes | yes | yes |

---

## the principled fix: transition cells

The correct approach is to replace the boundary cells on the coarse side with cells that have a different topology — cells that know about both resolutions simultaneously. Lengyel calls these <em>transition cells</em>, and the algorithm for generating them is <em>Transvoxel</em> [1].

### the seam geometry problem, restated

Consider any cell in the coarse-resolution block that lies along the face shared with the fine-resolution block. That cell has 8 corner voxel samples (the usual marching cubes input). But along its face touching the fine block, there are also 9 additional samples from the fine-resolution data: the 3×3 grid of fine samples covering the same face area. Those 9 samples know exactly where the fine mesh put its surface. If the cell can use all of this information — the 9 fine samples on one face plus the coarser samples on the opposite face — it can generate triangles that attach to the fine mesh on one side and to the rest of the coarse mesh on the other.

That is the transition cell. It is a cell that bridges two resolutions by seeing both simultaneously.

### what a transition cell looks like

A transition cell is divided into two parts along a plane parallel to the seam face [1]:

- The **full-resolution face** (toward the fine-resolution neighbor): a 2×2 grid of quadrants, each the size of one fine-resolution cell, carrying 9 sample points (a 3×3 grid of values at fine spacing).
- The **half-resolution face** (toward the interior of the coarse block): the normal 4 corner voxels shared with the coarse block's own marching cubes run. These 4 corners have the same density values as the 4 corners of the full-resolution face — they are the same spatial locations, just accessed by both faces.

```
Transition cell (schematic, face view):

  full-resolution face (9 samples):
  ┌──┬──┐
  │6 │7 │8
  ├──┼──┤  ← fine-resolution 3×3 grid
  │3 │4 │5
  ├──┼──┤
  │0 │1 │2
  └──┴──┘

  half-resolution face (4 corners only):
  B────C
  │         │  ← coarse 2×2 corners
  9────A

  Corners 0, 2, 6, 8 on the full-res face equal
  corners 9, A, B, C on the half-res face.
```

The left sub-cell (full-resolution face) is triangulated using only those 9 values and a lookup table, producing surface geometry that exactly matches the fine-resolution mesh's edge positions. The right sub-cell (half-resolution face) is triangulated with the ordinary modified marching cubes algorithm using the 8 coarse corner values, matching the rest of the coarse block's interior mesh.

The two sub-cells share their interior boundary, and the whole transition cell is therefore watertight with both neighbors.

### the lookup table

With 9 binary sample values on the full-resolution face (each either inside or outside solid space), there are 2⁹ = 512 possible configurations to triangulate. Lengyel applied the dihedral group D₈ (the symmetries of a square — 4 rotations and 4 reflections) to reduce these 512 cases to <em>73 equivalence classes</em>, analogous to how the 256 marching cubes cases reduce to 15 [1]. For each class, the correct triangulation is precomputed and stored, and a 512-entry lookup table maps any case code to its class and triangle pattern. At runtime, the algorithm reads the 9 sign bits, assembles a 9-bit index, fetches the class from the lookup table, and emits the triangles — the same O(1) structure as marching cubes.

Vertices are cached and reused across adjacent transition cells (using a history buffer covering the current and preceding row) to avoid redundant interpolation — the same technique used in the regular cell implementation.

### why the naive alternative is impractical

The most obvious approach would be to run a single large marching-cubes variant over the full set of samples at the boundary: 13 samples when one face is adjacent to a fine block (9 fine + 4 coarse-only), 17 when two adjacent faces are fine, 20 when three faces are fine. That gives 2¹³ + 2¹⁷ + 2²⁰ = 1,187,840 configurations to enumerate. Building and indexing a lookup table that large is impractical: it would require tens of megabytes of storage and produce enough cache misses to dominate per-cell rendering cost [1]. The transition cell design sidesteps this by restricting the sampling difference to exactly one factor of two and splitting the cell, keeping the lookup tables on the same order of magnitude as regular marching cubes.

### where transition cells go

Each coarse-resolution block generates up to six separate transition meshes — one for each face that might be adjacent to a finer-resolution block. At render time, the engine checks which of the block's six faces border a finer neighbor; only those transition meshes are rendered. If a neighbor is at the same resolution, no transition is needed on that face.

Because a transition cell occupies some volume (it has a nonzero width), the regular cells along the boundary edge of the coarse block must shrink slightly inward to make room. Each boundary vertex stores both a **primary position** (used when no transition is active on that face) and a **secondary position** (used when transition cells are rendered). The secondary position is computed by projecting the inward offset along the vertex's surface tangent plane, avoiding concavities [1]. The vertex program selects between them based on the runtime neighbor state.

---

## the other artifact: popping

Even with transition cells solving the crack problem, something still feels wrong when a chunk switches resolution level as the camera moves. The mesh abruptly changes shape — fewer triangles, different vertex positions — and the eye catches it as a pop. <em>Geomorphing</em> (or LOD morphing) addresses this.

The key insight Lengyel describes [1] is that every vertex on a low-detail mesh coincides exactly with a vertex on the highest-detail mesh — this is enforced during the low-resolution triangulation by always snapping to sub-cell positions that the high-resolution mesh would also use (Section 4.2.1 of the dissertation). Because of this coincidence, it is possible to store, for each vertex, not only its own position but also the position it would occupy if it belonged to the lower-detail mesh — its morph target.

At render time, the vertex shader lerps between these two positions based on a per-chunk blend factor `t` (0 = full detail, 1 = coarser detail). The blend factor is driven by camera distance: as a chunk moves toward its LOD switch threshold, `t` increases, smoothly morphing the mesh toward the coarser shape before the resolution actually changes. When the switch fires, `t` is already 1, and the pop is invisible.

Lengyel treats geomorphing as future work in his dissertation (Chapter 6.1) rather than a shipped feature, noting that storing secondary positions adds 12 bytes per vertex, which was a constraint on memory-limited platforms in 2010. Modern implementations — including the Godot voxel module — implement geomorphing via vertex shader attributes, encoding the secondary position as a compact per-vertex offset or direction [3].

### fade instead of morphing

A simpler alternative is alpha fade: render both the old and new LOD meshes simultaneously during the transition and blend their alpha over several frames. This avoids storing secondary positions but requires two draw calls and does not work for opaque geometry without dithered discard tricks. Most production engines prefer morphing for fully opaque terrain and reserve fading for distant impostors.

| technique | memory cost | transparent-safe | visual quality |
|---|---|---|---|
| no correction | zero | yes | bad (pop) |
| alpha fade | zero | no (without dither) | medium |
| geomorphing | ~12 bytes/vertex extra | yes | smooth |

---

## the engine side: lod scheduling

Transition cells and geomorphing solve the per-seam problem. The engine side must decide which blocks to render at which resolution and when to switch. The standard approach Lengyel describes [1] is to traverse a voxel octree during rendering:

- Each interior node of the octree covers the same world-space volume as 8 children at the next finer level.
- Blocks not intersecting the view frustum are culled along with their entire subtree.
- For each visible block, the engine projects the block's size into viewport space. If the projected size falls below a threshold, that block is rendered at its current resolution and its children are skipped. If the size is above threshold, the engine recurses into the children.
- The threshold can be tuned per platform to balance quality and GPU cost.

The result is that nearby blocks are always rendered at fine resolution, far blocks are rendered coarse, and transition cells cover every face that borders a resolution change — all in a single octree traversal.

For more detail on how engines integrate this system, see [LOD in engines](../engines/lod-in-engines.md). For the optimization perspective — why LOD matters for draw call budgets and GPU fill rates — see [LOD and culling](../optimization/lod-and-culling.md).

---

## transvoxel vs alternatives

Transvoxel is the dominant technique for watertight seam repair in smooth voxel terrain, but it is not the only one.

[Surface nets and dual contouring](./surface-nets-and-dual-contouring.md) place vertices in cell interiors rather than on edges. Dual contouring works naturally with octrees of varying cell size, and the seam mesh that connects different octree leaf sizes can be constructed without the full transition cell machinery — though getting it watertight across all topologies is non-trivial [4]. If the meshing algorithm is surface nets or dual contouring, Transvoxel's specific lookup tables do not apply.

See [choosing a meshing algorithm](./choosing-a-meshing-algorithm.md) for the full comparison of when each extraction method makes sense.

| seam approach | works with | watertight | complexity |
|---|---|---|---|
| skirts | any | no | low |
| Transvoxel | marching cubes / variants | yes | medium |
| DC seam mesh | dual contouring | yes | medium-high |
| stitching | height-field only | partial | low |

---

## the specifics

### sample numbering in a transition cell

Lengyel numbers the 13 sample locations in hexadecimal [1]:

- Full-resolution face (3×3 grid, 9 samples): **0–8**, laid out as:
  ```
  6  7  8
  3  4  5
  0  1  2
  ```
- Half-resolution face (4 corners): **9, A, B, C**, with 9 and A at the bottom, B and C at the top.
- The values at locations {0, 2, 6, 8} are identical to those at {9, A, B, C} because they are the same world-space points.

### the 9-bit case index

To construct the case index, each of the 9 full-resolution sample locations contributes one bit (its inside/outside sign). Lengyel assigns specific hex weights to each position so that the lowest two nibbles of the case code transpose correctly under 180° rotation, simplifying the lookup table structure. The resulting 9-bit index (range 0–511) directly addresses the transition lookup table.

### vertex secondary position formula

For a boundary vertex at position (x, y, z) in cell-local coordinates, the secondary position offsets (Δx, Δy, Δz) are computed by a piecewise linear function (Equation 4.2 in [1]) that scales the interior of boundary cells inward to create space for the transition layer. The width of the transition region is `w(k) = 2^(k−2)` where k is the LOD index. These offsets are then projected onto the tangent plane at the vertex (using the surface normal) to avoid flattening artifacts.

---

## references

[1] Lengyel, E. S. (2010). *Voxel-Based Terrain for Real-Time Virtual Simulations*. PhD dissertation, University of California, Davis. [local PDF](../papers/lengyel-2010-voxel-terrain-transvoxel.pdf) · [source](http://www.terathon.com/lengyel/Lengyel-VoxelTerrain.pdf)

[2] Procworld blog (2013). "Emancipation from the skirt." [source](http://procworld.blogspot.com/2013/07/emancipation-from-skirt.html) (Documents skirt z-fighting limitations and the switch to proper seam geometry.)

[3] Zylann (2023). *Godot Voxel — Smooth Meshing with Transvoxel*. Documentation. [source](https://voxel-tools.readthedocs.io/en/latest/smooth_terrain/) (Shows practical geomorphing implementation via vertex shader attributes and `lod_fade_duration`.)

[4] Gildea, N. (2014). "Dual Contouring: Seams & LOD for Chunked Terrain." *Nick's Voxel Blog*. [source](http://ngildea.blogspot.com/2014/09/dual-contouring-chunked-terrain.html) (Covers the seam mesh approach for dual contouring as an alternative to Transvoxel.)
