<link rel="stylesheet" href="./css/globals.css">

# voxel physics and destruction

Blow a hole through the base of a concrete pillar in Teardown and watch the slab above it tear free, tumble through the air, and come to rest as a pile of rubble. The debris stays exactly where it lands — the world is permanently changed. That single interaction is the benchmark this page is building toward: a voxel grid that can lose pieces, recognize which regions just became unattached, hand each one to a physics solver, and let it fall as a convincing rigid body.

Every part of that pipeline — detecting the break, computing the new body's mass and shape, running the solver, settling the debris — has a concrete technique behind it. This page covers each in turn.

---

## the coarse model — three stages

Before diving into mechanisms, the shape of the whole system:

1. **The edit happens.** An explosion, a tool, a collision — something removes voxels from the static grid. This is the CSG edit described in [runtime editing and CSG](../engines/runtime-editing-and-csg.md).
2. **Connectivity is re-checked.** The engine scans the affected region to find any solid clusters that are no longer joined to the ground or structure. Each disconnected cluster becomes a candidate for a new physics body.
3. **A rigid body is born.** The cluster gets a mass, a center of mass, an inertia tensor, and a collision shape. A physics solver takes it from there — gravity pulls it down, it tumbles and bounces, and eventually settles as static debris.

A reader who stops here has a true (if coarse) model. The rest of the page fills in how each stage actually works.

---

## detecting the break — connected-component analysis

### the core idea

After voxels are removed, the engine needs to answer one question: which groups of solid voxels are still connected to a fixed anchor (the ground, a wall mount, any "pinned" voxel), and which are now floating free?

The answer comes from treating the grid as a graph — each solid voxel is a node, and two voxels share an edge when they touch face-to-face. A group of solid voxels is connected if you can trace a path of face-adjacent voxels from any member to any other. Groups that contain at least one pinned voxel stay static. Groups that don't are now airborne debris.

This region-labeling process is called <em>connected-component analysis</em>. The practical algorithm that implements it is a 3D flood fill — start from every pinned voxel, spread through face-adjacency, mark everything reached. Whatever remains unmarked after the fill has lost its attachment to the world.

### the flood fill in plain language

Think of it as pouring dye into the structure at every grounded point simultaneously [1]. The dye seeps from voxel to voxel through every face-touching connection. After the fill completes, colored regions are attached; uncolored solid voxels are floating free. Each connected blob of uncolored voxels becomes one new rigid body.

```
// pseudocode — BFS flood fill from all anchors
queue ← all_pinned_voxels
visited ← empty set

while queue not empty:
    v ← queue.pop()
    for each face_neighbor n of v:
        if n is solid and n not in visited:
            visited.add(n)
            queue.push(n)

detached ← { v : v is solid and v not in visited }
// group detached into connected blobs → one rigid body each
```

The concept — spreading through face-adjacency from known-stable origins — comes first; this code is just its direct translation.

### 6-connectivity vs 26-connectivity

Two choices matter here:

| connectivity | what counts as a neighbor | implication |
|---|---|---|
| 6-connected (face) | the six ±X ±Y ±Z face-touching voxels | conservative — diagonal contact does not hold two pieces together |
| 26-connected (face + edge + corner) | all 26 touching voxels | permissive — a single corner voxel bridges two regions |

Most destruction systems use **6-connectivity** for structural attachment. It matches physical intuition: a block resting on a corner is not meaningfully supported. Using 26-connectivity would let voxel clusters cling together in unrealistic ways [1].

### performance — only re-analyze what changed

A brute-force flood fill over the entire world on every edit is far too expensive. The standard approach is to scope the analysis to the region an impact touched — a bounding box around the destroyed voxels plus a margin [2]. Only the voxels inside that region need re-labeling; the rest of the world stays cached.

Milan Bonten's voxel physics engine takes this further: the fill runs incrementally across multiple frames, budgeting a fixed number of voxels per frame [2]. Smaller objects resolve in one or two frames; a large 256³ structure might take 20–40 frames to fully re-analyze. This keeps frame time bounded at the cost of a short delay before large separations are noticed — acceptable in most destruction scenarios where the dramatic moment is the initial break, not the physics that follows a second later.

See [runtime editing and CSG](../engines/runtime-editing-and-csg.md) for how the edit itself is applied to the grid before this analysis begins.

---

## handing pieces to physics

### building the rigid body from a voxel blob

Once a detached cluster is found, it needs the physical properties a solver can act on. With a uniform voxel grid this is straightforward [1]:

- **mass** — each voxel carries a material tag; each material has a known density. Sum `density × voxel_volume` over every voxel in the cluster.
- **center of mass** — weighted average of each voxel's position by its mass contribution. For a uniform-material cluster this is just the geometric centroid.
- **inertia tensor** — treat each voxel as a small box; sum the parallel-axis contributions of each box around the cluster's center of mass. Because voxels are all the same size and axis-aligned, this is a loop, not a hard integral.

This is one of the genuine simplicity wins of a voxel representation: physical properties that require hand-tuning on a mesh fall out automatically from the grid.

### collision shape

A voxel blob needs a collision shape before a physics solver can test it against the world. Three common options, in order of cost and fidelity:

| shape | how | when to reach for it |
|---|---|---|
| axis-aligned bounding box | tightest-fitting box around the blob | fast but wrong for non-box shapes |
| convex hull | smallest convex envelope of the voxel centers | one rigid body, good for compact blobs |
| voxel-to-mesh | generate a surface mesh from the blob (blocky or greedy) and feed it to the solver | accurate for irregular shapes; see [blocky and greedy meshing](../meshing/blocky-and-greedy-meshing.md) |

In practice, small debris pieces use a bounding box or convex hull; large architectural chunks may get a quick surface mesh. The goal is "good enough to look right", not perfect accuracy — a falling chunk of wall does not need millimeter-precise collision.

### the physics solver

Once the blob is a rigid body, a standard solver takes over. Teardown uses a CPU-based custom solver built around **sub-stepping with Temporal Gauss-Seidel** instead of solver iteration [3]. The distinction matters for dense piles of debris: sub-stepping propagates impulses more reliably through a stack of hundreds of resting bodies than a single-step solver with many iterations. The Teardown solver runs on 32 threads in parallel — each thread handles a batch of non-interacting bodies — and achieves roughly 5 ms of simulation time per frame [3].

The solver's job: apply gravity, resolve collisions between falling debris and the static world, handle debris-on-debris stacking. It is the same solver described by Gustafsson in his early rigid body posts on the Voxagon blog [4].

### settling and re-integrating debris

A rigid body that has come to rest and is no longer moving can be put to sleep — taken out of the active solver set and treated as static geometry. This is the standard physics-engine sleep state.

Some systems go further: once debris settles, they re-merge it back into the static voxel grid entirely, so it no longer exists as a rigid body at all. This is the cheapest possible state — static voxels cost almost nothing compared to an active rigid body. The tradeoff is that re-merged debris can no longer be pushed or disturbed later. Whether to re-merge is a gameplay call; in a game like Teardown, debris that lands rarely needs to move again, so sleeping or re-merging is a reasonable win.

---

## realism extras

### structural integrity — unsupported structures that sag and collapse

The connected-component approach above treats all attached voxels as equally supported, regardless of geometry. A real arch over an empty doorway is supported; a 20-voxel horizontal beam attached only at one end should not hold its own weight.

A structural-integrity system models this. The standard game implementation propagates a **support value** outward from every grounded voxel, decaying with distance and horizontal span [5]. When a block's incoming support value falls below the weight of everything above it, it fails — and its failure reduces support for its neighbors, potentially cascading into a collapse.

Red Faction: Guerrilla popularized this model in games: each structural layer accumulates the weight of all layers above it, and when strength < weight, the layer fails [6]. 7 Days to Die implements the same idea as a propagated integer: blocks can only span a fixed number of voxels horizontally from a supported column before the overhang value exceeds the material limit and the block drops [5].

This is an approximation — it is not finite-element analysis — but it produces satisfying collapses: undercut a pillar and the floor above sags and drops, rather than floating in place until the last connecting voxel is severed.

### pre-fracture patterns

For high-detail destruction of objects like glass windows or vehicle panels, waiting for voxels to be individually removed produces a slow, blocky result. **Pre-fracture** sidesteps this: the object is authored with fracture seams already baked in as constraint lines. When the object takes damage above a threshold, the constraints break and the pre-fractured shards fly apart as individual rigid bodies.

The two common pre-fracture layouts are:

- **Voronoi fracture** — seed points scattered through the object volume; each fragment is the Voronoi cell around one seed. Produces organic-looking irregular shards.
- **Grid fracture** — regular rectangular cuts, faster to compute, looks more artificial but works well for concrete or masonry.

Pre-fracture is not specific to voxels — it is widely used on meshes too — but it maps naturally onto a voxel grid because each fragment can be a cluster of voxels separated by a seam of empty cells.

---

## performance — keeping it real-time

Teardown's approach to performance is architectural as much as algorithmic [7]:

- **Keep the world small.** Each Teardown level fits within a ~2,504 × 256 × 2,504 voxel envelope, where each voxel is 10 cm — roughly a 250 m × 25 m footprint [8]. A tighter world means fewer voxels to flood-fill, fewer rigid bodies to simulate, and a renderer that can hold the entire world in a compact representation.
- **Thousands of small volumes, not one massive grid.** Rather than one giant voxel array, the engine stores objects as individual voxel volumes. Each volume is an independent, axis-aligned block. Physics and connectivity analysis run per-object, not across the whole scene [7].
- **CPU for collision, GPU for rendering.** Voxel-to-voxel contact generation runs on the CPU, which is natural for the irregular, dynamic queries destruction requires. Rendering is GPU-handled via ray tracing in the fragment shader [7]. The two pipelines are largely decoupled.
- **Localized re-analysis.** As noted above, flood-fill after an edit covers only the impacted bounding region. See [dense grids and chunks](../storing/dense-grids-and-chunks.md) for how the grid is partitioned to make this region lookup fast.
- **Parallel solver.** The rigid-body solver distributes non-interacting body groups across threads — details in the Teardown physics engine showcase [3].

The [case studies](../engines/case-studies.md) page examines how these choices compose into the full Teardown engine.

---

## putting it together — the full pipeline

One destruction event, end to end:

1. An explosion calls the CSG edit function — voxels in a sphere are written to empty. ([runtime editing and CSG](../engines/runtime-editing-and-csg.md))
2. A bounding box around the destroyed region is queued for connectivity re-analysis.
3. A flood fill runs from all grounded/pinned voxels in that bounding box.
4. Any solid voxel not reached by the fill is collected; connected blobs of unreached voxels are grouped.
5. Each blob computes its mass, center of mass, inertia tensor, and collision shape.
6. The blob is handed to the rigid-body solver as a new dynamic body.
7. The solver applies gravity; the chunk falls, bounces, tumbles.
8. When velocity drops below a threshold the body sleeps or is re-merged into the static grid.

The grid that the simulation runs on — and why a regular array is the right structure for this — is described in [voxels as a simulation grid](./voxels-as-a-simulation-grid.md).

---

## references

[1] Procworld Blog. (2013, December). "Voxel Physics." Procedural World. Technical devlog on flood-fill connectivity, mass from voxel density, and mesh-based fragment physics. [source](http://procworld.blogspot.com/2013/12/voxel-physics.html)

[2] Bonten, M. (n.d.). "Voxel Physics Engine." Milan Bonten portfolio. Implementation notes covering flood fill across multiple frames, 1-byte voxel physics metadata, SAT early-out. [source](https://milanbonten.github.io/voxel-physics-engine)

[3] 80.lv. (2024). "CPU-Based Custom Voxel Physics Engine." 80 Level. Coverage of Teardown creator's new physics solver: Temporal Gauss-Seidel sub-stepping, 32-thread parallel solver, ~5 ms simulation time, contact generation and broad-phase improvements. [source](https://80.lv/articles/see-what-s-new-in-teardown-creator-s-custom-voxel-physics-engine)

[4] Gustafsson, D. (2010). "Explaining the rigid body solver." Voxagon Blog. Foundational post on impulse-based rigid body solvers by the Teardown creator. [source](https://blog.voxagon.se/)

[5] 7 Days to Die Wiki contributors. (2024). "Structural Integrity." 7 Days to Die Wiki. Documents the propagated support-value model: material load limits, overhang distance caps, and cascading collapse. [source](https://7daystodie.fandom.com/wiki/Structural_Integrity)

[6] Brown, M. (2022). "How Games Do Destruction." GMTK Substack. Survey of destruction techniques in games including Red Faction Guerrilla's layer-stress model, Teardown's loose-voxel-to-rigidbody pipeline, and pre-fracture approaches. [source](https://gmtk.substack.com/p/how-games-do-destruction)

[7] Gustafsson, D. / Tuxedo Labs. (2020). "How beautiful voxels laid the way for Teardown's heist-y framework." Game Developer. Covers the architecture: thousands of small voxel volumes, CPU collision, GPU rendering, level-size as a deliberate performance constraint. [source](https://www.gamedeveloper.com/design/how-beautiful-voxels-laid-the-way-for-i-teardown-s-i-heist-y-framework)

[8] JuanDiegoMontoya. (n.d.). "Teardown Breakdown." Technical rendering analysis. Documents the 1252×128×1252 world texture (2504×256×2504 effective voxels at 10 cm each) and three-level mip hierarchy for ray traversal. [source](https://juandiegomontoya.github.io/teardown_breakdown.html)

[9] 80.lv. (2024). "Teardown Developer Breaks Down Multiplayer and Voxel Destruction Tech." 80 Level. Gustafsson on voxel volumes on regular grids, SIMD and multithreading for collision, deterministic command replication for multiplayer. [source](https://80.lv/articles/teardown-developer-breaks-down-multiplayer-and-voxel-destruction-tech)
