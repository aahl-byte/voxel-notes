<link rel="stylesheet" href="./css/globals.css">

# marching cubes

A CT scanner finishes its sweep and the radiologist wants to see the patient's femur as a solid 3D model she can rotate and inspect. A game engine has generated a density field for a chunk of terrain and needs a triangle mesh the GPU can render at 60 fps. A visual effects artist has simulated a smoke plume as a grid of density values and wants to extract the visible surface so it can be lit and shaded.

All three are the same problem: *given a 3D scalar field — a grid of numbers, one per cell — extract the surface where the field crosses a chosen threshold.* That threshold is the <em>iso-value</em>, and the surface it defines is the <em>iso-surface</em>.

Marching cubes solves this by processing one grid cell at a time, looking up what triangles to emit for each possible corner configuration, and outputting a triangle mesh that approximates the iso-surface across the entire field. It is fast enough to run at interactive rates, simple enough to implement in an afternoon, and was published by Lorensen and Cline at SIGGRAPH in 1987 [1]. It became one of the most cited papers in computer graphics, and most volume-to-mesh pipelines still use it or a close descendant.

For background on why you'd want a mesh from a voxel field at all, see [why mesh voxels](./why-mesh-voxels.md). For context on what scalar fields a voxel volume can store, see [voxel data models](../foundations/voxel-data-models.md).

---

## the coarse model — what the algorithm actually does

Here is the whole idea in one paragraph, before any detail.

Imagine walking through a 3D grid, picking up one cube-shaped cell at a time. Each cube has eight corners. Each corner has a scalar value from the field. You pick a threshold — the iso-value — and for each corner you ask: is this value above or below the threshold? That gives you eight yes/no answers, one bit each, forming an 8-bit number from 0 to 255. You look that number up in a precomputed table. The table tells you which edges of the cube the surface crosses, and in what triangle pattern. You place each triangle vertex on its edge by interpolating between the two endpoint values to find exactly where the field equals the iso-value. You emit those triangles. Then you move to the next cube. When you have visited every cube, the collected triangles form a mesh that approximates the iso-surface.

That is <em>marching cubes</em> — a per-cell lookup table approach to iso-surface extraction.

A beginner's summary:

- **Input:** a 3D scalar field (a grid of numbers) and a threshold value.
- **Output:** a triangle mesh approximating the surface where the field equals the threshold.
- **Mechanism:** for each grid cell, classify corners as inside/outside → look up triangle configuration → interpolate vertex positions → emit triangles.
- **Cost:** linear in the number of cells — every cell is visited once.

---

## the algorithm, step by step

### step 1 — classify each corner

For a cell at grid position (i, j, k), the eight corners are the eight neighbouring grid points: (i, j, k), (i+1, j, k), (i, j+1, k), and so on. Each has a scalar value from the field.

For each corner, compare its value to the iso-value:

- value ≥ iso-value → corner is **inside** (bit = 1)
- value < iso-value → corner is **outside** (bit = 0)

Assign one bit per corner. The eight bits, assembled in a fixed order, form an 8-bit integer — the <em>case index</em>. Since there are 8 bits, there are 2⁸ = 256 possible case indices.

### step 2 — look up the triangle configuration

A precomputed table maps each of the 256 case indices to a list of which edges of the cube the surface intersects, and how those intersection points are connected into triangles. The table was computed once, offline, by hand or by enumeration.

The 12 edges of a cube are numbered 0–11. Each table entry is a list of edge triples: `[edge_a, edge_b, edge_c, edge_d, ...]` where each consecutive triple forms one triangle.

Case index 0 (all corners outside) and case index 255 (all corners inside) both produce an empty list — the surface doesn't pass through the cell at all. All other indices produce one to five triangles.

### step 3 — place vertices by interpolating along edges

For each edge the surface crosses, find the two corner values at its endpoints and the iso-value, then interpolate linearly:

```
t = (iso_value - value_A) / (value_B - value_A)
vertex = position_A + t * (position_B - position_A)
```

`t` is the fraction along the edge where the field equals the iso-value. When `t = 0` the surface grazes corner A; when `t = 1` it grazes corner B; values in between place the vertex proportionally. This is the step that gives marching cubes its smooth appearance — the vertex is not snapped to a grid point but placed exactly where the interpolated field crosses the threshold.

### step 4 — emit triangles and march on

The interpolated edge vertices, connected in the triples specified by the lookup table, form the triangles for this cell. Add them to the output mesh. Then advance to the next cell — the algorithm marches through the entire grid, visiting each cell exactly once.

---

## the 256 cases collapse to 15

A lookup table with 256 entries is manageable. What makes it elegant is that most of those 256 entries are just rotations or reflections of the same few patterns.

Consider what determines the surface shape inside a cube: only which corners are inside and which are outside. Two configurations that are related by rotating the cube produce the same surface shape, just rotated. Two configurations where you swap which corners are inside and which are outside produce a complementary surface (the same patch, seen from the other side).

Applying rotational symmetry, reflective symmetry, and complementarity, Lorensen and Cline showed that all 256 configurations reduce to just **15 topologically distinct base cases** [1]. The full 256-entry table is built by taking each base case and generating all its rotations and reflections.

The 15 cases range from the trivial (no surface, or a single triangle clipping a corner) to the complex (five triangles filling the cell with a saddle-shaped patch). Knowing there are only 15 base shapes is useful for reasoning about the algorithm and for spotting bugs — if a generated surface has unexpected holes or spikes, one of the 15 cases is likely misimplemented.

---

## the ambiguity problem — and three fixes

The 15 base cases look complete, but the original algorithm has a flaw discovered shortly after publication: in certain configurations, the correct way to draw the triangles inside a cell is **not uniquely determined** by knowing only which corners are inside and outside. This leads to holes in the output mesh.

### what ambiguity means

Take a cube face where the four corner values alternate: two diagonally opposite corners are inside, two are outside. The iso-surface enters through two edges and exits through two edges on that face. But which inside corner connects to which outside corner? There are two valid-looking interpretations — and if the two cells that share that face each pick a different interpretation, a gap opens between them.

This is <em>face ambiguity</em>. There is also <em>interior ambiguity</em>: even when all faces are consistent, the topology of the patch inside the cube can be ambiguous — the surface could connect the inside regions as a tube or as two separate sheets, and the 8-bit case index cannot distinguish between them.

Both types of ambiguity produce the same visible symptom: holes or tunnels in an otherwise smooth mesh.

### fix 1 — the asymptotic decider

Nielson and Hamann (1991) observed that the correct interpretation on an ambiguous face can be determined by treating the scalar field on that face as a bilinear function of the two face coordinates and finding where the interpolant's zero crossings actually lie [2].

The isoline on a bilinear face follows a hyperbola. The two branches of that hyperbola determine which corners connect. The algorithm evaluates the field at the hyperbola's asymptotic centre — the saddle point — and compares it to the iso-value. If the saddle is above the iso-value, the two inside corners connect; if below, they separate. This is the <em>asymptotic decider</em>: a single value comparison that resolves face ambiguity without guesswork.

The asymptotic decider resolves face ambiguities correctly but does not address interior ambiguities — cases where the cube's inner topology is still underdetermined even after all faces are resolved consistently.

### fix 2 — marching cubes 33

Chernyaev (1995) took a more comprehensive approach [3]. Rather than patching the original 15-case table, he enumerated all topologically distinct configurations of the trilinear interpolant inside a cube — the full scalar field, not just the corner signs. He found **33 topologically distinct cases** (instead of 15), accounting for both face and interior ambiguities. The extended lookup table, combined with the asymptotic decider for face disambiguation, produces a topologically correct mesh with no holes or tunnels for any input field.

The 33-case table is larger and the case selection logic is more involved, but the output is provably correct under the trilinear interpolation model. Lewiner et al. (2003) later published a clean, efficient implementation [4].

### fix 3 — marching tetrahedra (sidestep the problem entirely)

Both fixes above patch the cube-based approach. There is a simpler alternative: split each cube into six tetrahedra and run a marching algorithm on those instead.

Each tetrahedron has only four corners, so there are only 2⁴ = 16 possible configurations. Of those, most are trivial. Crucially, **no configuration is ambiguous** — four corners with defined inside/outside states always determine the surface patch uniquely, with no room for a hyperbolic branching decision. Marching tetrahedra is unambiguous by construction [5].

The trade-off:

| | marching cubes | marching tetrahedra |
|---|---|---|
| cases | 256 (15 base) | 16 per tet × 6 tets per cube |
| ambiguity | face + interior | none |
| triangle count | lower | higher (roughly 3–4× more per cube) |
| mesh quality | good triangle shapes | weaker aspect ratios, interpolation along face diagonals creates slight bumps |
| sharp features | rounded | also rounded |

When to reach for which:
- **Marching cubes + MC33** — when triangle count matters and you want the established workhorse with correctness guarantees.
- **Marching tetrahedra** — when simplicity of implementation is the priority, you need provably hole-free output, and the higher triangle count is acceptable.
- **Original marching cubes (no fix)** — only for previews or cases where occasional holes in the mesh are tolerable.

---

## normals from the gradient

Marching cubes produces vertices and triangles but not vertex normals. For smooth shading you need normals. There are two approaches.

**Face normals** — compute the cross product of each triangle's two edge vectors. Fast, but produces faceted shading (each triangle is flat-shaded) and adjacent triangles can have visibly different normals.

**Gradient normals** — the scalar field has a gradient at every grid point: the rate of change in each direction. The gradient is perpendicular to the iso-surface by definition. So the gradient of the field *at a corner* is the surface normal *at that corner*. Estimate it with finite differences:

```
gradient_x = (field[x+1, y, z] - field[x-1, y, z]) / (2 * cell_size)
gradient_y = (field[x, y+1, z] - field[x, y-1, z]) / (2 * cell_size)
gradient_z = (field[x, y, z+1] - field[x, y, z-1]) / (2 * cell_size)
```

Then, for each triangle vertex on a cell edge, linearly interpolate the gradient between the two corner gradients using the same `t` value used for the vertex position. Normalize the result. This produces smooth, per-vertex normals that respect the field's curvature rather than the triangle's shape, and it is essentially free — the same interpolation pass that places vertices can carry the gradients along.

Gradient normals are strongly preferred over face normals for any rendering that needs smooth appearance. For [volume ray casting](../rendering/volume-ray-casting.md), the gradient is the lighting normal used in every fragment.

---

## cost, limitations, and what comes next

### computational cost

Marching cubes is O(N) in the number of voxels — each cell is visited once. In practice, an unoptimized CPU implementation processes a 256³ grid in tens to hundreds of milliseconds. GPU implementations using compute shaders can run each cell in parallel and process the same grid in single-digit milliseconds, making real-time extraction practical for [procedural terrain](../generating/procedural-terrain.md) that changes as the player moves.

### the sharp-edge problem

The smoothness that makes marching cubes attractive is also its main limitation. Because vertex positions are constrained to lie on cell edges, the mesh cannot represent sharp corners or creases — a sharp 90° edge in the scalar field becomes a rounded curve in the triangle mesh, at a radius roughly equal to the cell size. No amount of field resolution eliminates this; it is structural.

For geometry that requires sharp features — architectural models, mechanical parts, terrain with hard cliff edges — [surface nets and dual contouring](./surface-nets-and-dual-contouring.md) place vertices inside cells rather than on edges, and use the field gradient to snap them to sharp features. The trade-off is higher implementation complexity and weaker topological guarantees.

### the algorithm's place in the pipeline

Marching cubes is the standard first choice for surface extraction because it is fast, simple, and produces well-shaped triangles. Its weaknesses are sharp features and, without MC33, topological holes. The [choosing a meshing algorithm](./choosing-a-meshing-algorithm.md) page walks through the decision: when marching cubes is the right call, when dual contouring earns its complexity, and when blocky or greedy meshing sidesteps the problem entirely.

At large scales, a single marching cubes pass over a high-resolution field is too expensive. Combining marching cubes with level-of-detail requires carefully stitched borders between coarse and fine cells — that seam problem and its standard solution are covered in [LOD seams and transvoxel](./lod-seams-and-transvoxel.md).

---

## references

[1] Lorensen, W. E. and Cline, H. E. (1987). "Marching cubes: A high resolution 3D surface construction algorithm." *ACM SIGGRAPH Computer Graphics*, 21(4), 163–169. DOI: 10.1145/37401.37422. (ACM-paywalled; the original paper introducing the algorithm and the 15-case table.)

[2] Nielson, G. M. and Hamann, B. (1991). "The asymptotic decider: resolving the ambiguity in marching cubes." *Proceedings of IEEE Visualization '91*, pp. 83–91. DOI: 10.1109/visual.1991.175782. [local PDF](../papers/nielson-hamann-1991-asymptotic-decider.pdf) · [source](https://escholarship.org/uc/item/17p025zk)

[3] Chernyaev, E. V. (1995). "Marching cubes 33: Construction of topologically correct isosurfaces." CERN Technical Report CN/95-17. [local PDF](../papers/chernyaev-1995-marching-cubes-33.pdf) · [source](https://repository.cern/records/7zfxg-q0t96)

[4] Lewiner, T., Lopes, H., Vieira, A. W., and Tavares, G. (2003). "Efficient implementation of marching cubes' cases with topological guarantees." *Journal of Graphics Tools*, 8(2), 1–15. DOI: 10.1080/10867651.2003.10487582. (Clean, correct MC33 implementation; the paper most practitioners use as their reference.)

[5] Doi, A. and Koide, A. (1991). "An efficient method of triangulating equi-valued surfaces by using tetrahedral cells." *IEICE Transactions on Information and Systems*, E74-D(1), 214–224. (Original marching tetrahedra paper.)

[6] Newman, T. S. and Yi, H. (2006). "A survey of the marching cubes algorithm." *Computers & Graphics*, 30(5), 854–879. DOI: 10.1016/j.cag.2006.07.021. [local PDF](../papers/newman-yi-2006-survey-marching-cubes.pdf) · [source](https://cgl.ethz.ch/teaching/scivis_common/Literature/Newman06.pdf) (Comprehensive survey covering the algorithm, all major extensions, and ambiguity resolutions.)
