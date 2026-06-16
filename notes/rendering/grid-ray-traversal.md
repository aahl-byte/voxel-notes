<link rel="stylesheet" href="./css/globals.css">

# grid ray traversal

Every meaningful thing a voxel renderer does — casting a primary ray from the camera, testing whether a point is in shadow, gathering ambient occlusion samples, marching a cone for global illumination, or simply finding the first solid voxel the player is looking at — requires the same operation: starting from some position in space and walking forward along a direction until a filled cell is found. The grid has to be visited cell-by-cell, in order, without skipping a cell the ray actually crosses and without visiting one it doesn't.

This page is about how to do that walk correctly and cheaply.

---

## the coarse model

The grid is divided into equal-sized cells along each axis. A ray enters the grid at some point and exits at another. As it travels, it crosses cell boundaries — first an X boundary, then maybe a Y boundary, then an X boundary again, and so on. Each time the ray crosses a boundary it moves into a new cell.

The insight behind the algorithm: at any moment, the ray is about to cross one of three possible boundaries — the next X plane, the next Y plane, or the next Z plane. Whichever boundary is *closest along the ray* is the one crossed first. Step across that boundary into the next cell, update which boundary comes next on that axis, and repeat.

That's the whole loop. Nothing more complicated than "pick the nearest upcoming wall and walk through it."

This is <em>DDA — Digital Differential Analyzer</em> applied to 3D grids, formalized for ray tracing by Amanatides and Woo in 1987 [1]. The paper's title — "A Fast Voxel Traversal Algorithm for Ray Tracing" — names the canonical algorithm. Everything below is unpacking how it works and why it is both exact and cheap.

---

## why exactness matters

Before the mechanics, it's worth understanding why a naive alternative fails.

The classic algorithm for drawing lines on a pixel grid is <em>Bresenham's line algorithm</em>, which advances along the dominant axis one integer step at a time. It is fast and produces clean results for rasterization. But for ray-grid traversal it has a defect: when a ray travels nearly diagonally, Bresenham can step only along X or only along Y at a corner where the ray actually clips both cells. A cell the ray physically passes through can be silently skipped.

For rendering that is unacceptable. A skipped cell means a missed surface — shadows that don't cast, objects the camera sees through, ambient occlusion that samples the wrong geometry.

The DDA approach avoids this entirely because it tracks progress along *all three axes simultaneously*. Every time the ray crosses a cell boundary it steps exactly once on the axis that boundary belongs to. Because every grid boundary the ray crosses triggers exactly one step, no cell the ray traverses can be missed, and no cell it doesn't traverse is ever visited. The traversal is exact by construction [1].

---

## the moving parts

### initialization — setting up tMax and tDelta

Before the loop runs, the algorithm computes two sets of per-axis values: <em>tMax</em> and <em>tDelta</em>.

Think of the ray as parameterized by a scalar `t`. At `t = 0` the ray is at its origin; at `t = 1` it is one unit away in the direction of the direction vector. The grid's cell boundaries are planes at fixed positions along each axis.

**tDelta** is the distance along the ray — measured in units of `t` — between consecutive parallel boundaries on one axis. If the ray travels steeply in X and shallowly in Y, `tDeltaX` will be small (boundaries come quickly) and `tDeltaY` will be large (boundaries come slowly). For unit-sized voxels it is simply `1 / |direction|` per axis.

**tMax** is the `t` value at which the ray hits the *very next* boundary on each axis from the current cell. At initialization this is computed from the ray origin's fractional position within the starting cell and the direction sign:

- if `direction.x > 0`: the next X boundary is at `cell_x + 1`
- if `direction.x < 0`: the next X boundary is at `cell_x`
- `tMax.x = (boundary_x_position - origin.x) / direction.x`

The same calculation applies to Y and Z. The result is three values — `tMax.x`, `tMax.y`, `tMax.z` — each telling you how far along the ray until that axis's next wall.

**step** is simply the sign of each direction component: `+1` or `-1` per axis. It determines which neighbor to move into when crossing a boundary on that axis.

### the traversal loop

Once initialized, the loop is three comparisons and one update per iteration:

1. Find the axis whose `tMax` is smallest — that is the next boundary the ray reaches.
2. Step the integer cell coordinate on that axis by `step[axis]`.
3. Add `tDelta[axis]` to `tMax[axis]` (the next boundary on that axis is now one cell further away).
4. Visit the new cell. If it is occupied, stop. Otherwise, continue.

In pseudocode (after Amanatides & Woo [1]):

```
// initialization
step = sign(dir)
tDelta = abs(voxelSize / dir)         // per axis
tMax   = (nextBoundary - origin) / dir // per axis, first boundary from start cell
cell   = floor(origin / voxelSize)     // starting cell

// traversal loop
while cell inside grid:
    if tMax.x < tMax.y:
        if tMax.x < tMax.z:
            cell.x += step.x
            tMax.x += tDelta.x
        else:
            cell.z += step.z
            tMax.z += tDelta.z
    else:
        if tMax.y < tMax.z:
            cell.y += step.y
            tMax.y += tDelta.y
        else:
            cell.z += step.z
            tMax.z += tDelta.z
    if grid[cell] is occupied:
        return cell, t_of_hit
```

That inner body is two comparisons and one addition on the floating-point side, plus one integer addition and one bounds check. Per step, it is as cheap as grid traversal can get.

### what the hit surface normal is — for free

The axis that was stepped on the iteration that found a hit tells you the surface normal of the hit face: step on X → normal is `(±1, 0, 0)`, step on Y → `(0, ±1, 0)`, step on Z → `(0, 0, ±1)`. No extra computation needed.

The parametric `t` of the hit is the `tMax` value on the stepped axis *before* the final increment — or equivalently `max(tMax.x, tMax.y, tMax.z)` after stepping. World-space hit position is `origin + t * dir`.

---

## why the flat-grid DDA is exact and cheap — the contrast

| property | Bresenham (3D) | DDA / Amanatides-Woo |
|---|---|---|
| missed cells | yes — diagonal corners | never |
| per-step work | integer arithmetic only | 2 float comparisons + 1 float add |
| surface normal | requires post-processing | free from stepped axis |
| error accumulation | integer, no float drift | float tMax grows additively — stable [1] |
| used for | rasterization, voxelization | ray-grid traversal |

Use DDA for ray traversal. Bresenham is the right tool for voxelizing lines and curves onto a grid — not for casting rays through one.

---

## making it fast on large or sparse grids

The flat-grid DDA visits every cell the ray crosses, one by one. On a 512³ grid, a ray through a mostly empty scene might visit hundreds of empty cells before hitting anything. That is wasteful.

Two strategies address this.

### empty-space skipping with a distance field

If every empty cell in the grid stores the distance to the nearest solid cell — a <em>voxel distance field</em> — then at each step the traversal can look up how far the ray can travel without hitting anything and jump that many cells forward instead of one. The result is a hybrid: DDA-style boundary tracking but with variable jump sizes in empty regions, similar in spirit to [sphere tracing](../generating/sdf-and-csg-modeling.md) but operating on discrete grid samples rather than a continuous SDF.

This is the fastest known approach for a flat dense grid — one lookup per step, maximum empty-space skipping [2].

### hierarchical traversal — multiple DDA scales

A more general solution is to layer the grid into a hierarchy: coarse cells covering 8 or 64 fine cells, and so on. The traversal runs a DDA at the coarse level and only drops to the fine level when the coarse cell is occupied. Empty coarse cells are crossed in a single step regardless of how many fine cells they contain.

This is what VDB's <em>Hierarchical DDA (HDDA)</em> does: it runs parallel DDAs at each level of the VDB tree and uses the structure to skip entire empty subtrees in one step [3]. The algorithm for [sparse voxel octree raytracing](./sparse-voxel-octree-raytracing.md) is a specialization of this idea — the octree is a hierarchy and traversal descends only into occupied children.

Hierarchical grids can outperform octrees on raw traversal speed because they avoid pointer indirection: each level is a regular array, addressable by index arithmetic rather than pointer chasing [2].

---

## the GPU picture

On the GPU, every pixel fires one primary ray. Those rays are grouped into 32-thread warps, and within a warp all threads execute the same instruction. This is where ray coherence matters.

**Primary rays** from neighboring pixels point in nearly the same direction and hit roughly the same region of the scene. Within a warp, threads advance through the DDA loop at nearly the same rate and step the same axis at the same iteration most of the time. Warp divergence — threads taking different branches — is low, and the traversal runs close to peak throughput [4].

**Shadow rays** fired from a lit surface point toward the same light source, so they also converge geometrically and behave similarly to primary rays in terms of coherence. Ambient occlusion rays and GI cone samples are less coherent (scattered directions) and suffer more warp divergence, which is why those effects are more expensive per sample.

Practical GPU voxel renderers dispatch one thread per pixel, accumulate the DDA loop in a shader, and use shared memory or L1 cache to amortize repeated grid reads when neighboring threads access the same region.

---

## contrast: DDA vs sphere tracing

<em>Sphere tracing</em> (Hart 1996 [5]) is the other main ray-advancing strategy used in voxel-adjacent rendering. The comparison is instructive.

| | DDA / fast voxel traversal | sphere tracing |
|---|---|---|
| data required | occupied/empty flags per cell | continuous SDF value per point |
| step size | fixed — one cell boundary at a time | variable — the SDF value at the current point |
| guarantee | visits every cell the ray crosses, in order | never overshoots a surface (safe-step guarantee) |
| empty-space behavior | one step per cell (unless hierarchical) | large steps in open space, tiny near surfaces |
| storage | discrete voxel grid | dense or sparse SDF |
| best for | discrete voxel grids, exact hit cells, block-style geometry | smooth implicit surfaces, procedural SDFs, [SDF/CSG modeling](../generating/sdf-and-csg-modeling.md) |

Sphere tracing cannot be used on a raw voxel occupancy grid because it requires a smooth distance function; DDA cannot efficiently render a smooth SDF without first discretizing it. When voxels *store* distance values (a voxel SDF), the two ideas merge: use DDA to march cell-to-cell and use the stored distance value to decide how many cells to skip — the hybrid approach described under empty-space skipping above.

The choice between them is really a choice of representation. Discrete geometry → DDA. Smooth implicit fields → sphere tracing. Overlapping cases can use hybrid strategies.

---

## where this sits in the rendering pipeline

Grid ray traversal is the inner loop of almost every technique covered in this domain.

- [ways to render voxels](./ways-to-render-voxels.md) surveys which render strategies need this loop and which sidestep it entirely.
- [sparse voxel octree raytracing](./sparse-voxel-octree-raytracing.md) extends DDA into a tree, descending into occupied octants rather than advancing one flat cell at a time.
- [choosing a render path](./choosing-a-render-path.md) puts traversal cost in context alongside rasterization and splatting alternatives.
- [gpu voxel techniques](../optimization/gpu-voxel-techniques.md) covers the memory layout choices — brickmaps, VDB, SVO — that determine how fast each grid lookup inside the loop actually is.

The grid that traversal walks through is described in [the voxel grid](../foundations/the-voxel-grid.md).

---

## the specifics

### termination

The loop exits when:
- the cell coordinate on any stepped axis moves outside the grid bounds (`cell.x < 0` or `cell.x >= grid.width`, etc.) — the ray has exited the grid without hitting anything;
- a cell is found to be occupied — hit, return cell coordinates and `t`;
- a maximum `t` is reached — useful for shadow rays (if nothing is hit within the light distance, the point is lit) and ambient occlusion (sample up to some radius).

### handling rays parallel to an axis

If `direction.x == 0`, the ray never crosses an X boundary. Set `tMax.x = +infinity` and `tDelta.x = +infinity`. The X comparison in the loop will always lose, and X will never be stepped. The same applies to Y and Z. This makes zero-direction components a non-case — the loop handles them without special branches.

### numerical stability

`tMax` values are computed once at initialization and then only incremented by `tDelta`. Because `tDelta` is constant per axis, the additions accumulate floating-point rounding over many steps. For deep traversals (thousands of steps), error can build up enough to mis-identify which axis should be stepped. Robust implementations either use double precision for tMax/tDelta or periodically recompute tMax from integer cell coordinates rather than accumulating indefinitely.

### grid entry

The algorithm starts from a cell index and a set of initial tMax values. If the ray origin is *outside* the grid, the entry point must be found first — intersect the ray with the grid's axis-aligned bounding box (a standard slab intersection), clamp `t` to the entry point, and derive the entry cell from `origin + t_entry * dir`. The DDA then picks up from there.

---

## references

[1] Amanatides, J. and Woo, A. (1987). "A Fast Voxel Traversal Algorithm for Ray Tracing." *Eurographics '87*, Eurographics Association. DOI: 10.2312/egtp.19871000. [local PDF](../papers/amanatides-woo-1987-fast-voxel-traversal.pdf) · [source](http://www.cse.yorku.ca/~amana/research/grid.pdf)

[2] DubiousConst282 (2024). "A guide to fast voxel ray tracing using sparse 64-trees." Technical blog. [source](https://dubiousconst282.github.io/2024/10/03/voxel-ray-tracing/)

[3] Museth, K. (2013). "VDB: High-Resolution Sparse Volumes with Dynamic Topology." *ACM Transactions on Graphics*, 32(3), Article 27. DOI: 10.1145/2487228.2487235. [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf) · [source](https://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf)

[4] Braley, C. and Sandu, A. (2010). "GPU Accelerated Voxel Traversal using the Prediction Buffer." *Eurographics Symposium on Parallel Graphics and Visualization*. [source](https://people.cs.vt.edu/~yongcao/publication/pdf/Braley10.pdf)

[5] Hart, J. C. (1996). "Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces." *The Visual Computer*, 12(10), 527–545. DOI: 10.1007/s003710050084. [source](https://link.springer.com/article/10.1007/s003710050084)
