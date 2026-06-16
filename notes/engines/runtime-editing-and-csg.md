<link rel="stylesheet" href="./css/globals.css">

# runtime editing and CSG

A player digs a tunnel through rock and the tunnel stays dug. A tank shell punches through a wall and the wall stays punched. A sculptor drags a sphere brush across terrain and the landscape deforms under it, live, at sixty frames per second. An editor user carves a cave system by painting a subtract brush, then hits undo and it un-carves.

These are all the same underlying operation: code writes new values into voxel cells at runtime, and the visible world updates to match. The world is writable, not baked — and that writability is what separates a voxel world from a static mesh scene.

Making it work correctly and cheaply is the topic of this page. The three pillars are: **what the edit operation actually does** to the data, **how the engine propagates the change** to only the meshes that need it, and **how you record edits** so they can be undone, replayed, or sent to other players. For how runtime editing fits into the full engine architecture alongside generation, streaming, and rendering, see [anatomy of a voxel engine](./anatomy-of-a-voxel-engine.md).

---

## what an edit operation does

### writing cells — blocky voxels

In the simplest case — a blocky, Minecraft-style world stored as a flat array of material IDs — an edit is a write loop. A brush defines a shape (sphere, box, path) and a target value. The loop visits every cell whose position falls inside that shape and sets its material channel to the target value.

```python
# Place stone sphere, radius 4, centered at world position p
for each cell (x, y, z) within AABB of sphere:
    if distance(cell_center, p) <= 4.0:
        grid[x, y, z].material = STONE   # MODE_SET
```

To dig, set the target to AIR (or whatever your empty sentinel is). To place, set it to the desired material ID. The data change is trivially local: only cells inside the brush shape change. This is one of the fundamental wins of the voxel representation — a boolean edit is just a write to cells, not a mesh surgery operation.

Bulk functions (`do_sphere`, `do_box`, `do_path`) are always preferred over per-cell calls in production engines. They let the engine batch the data locking, cache data-structure lookups, and schedule exactly one dirty-mark per affected chunk rather than one per cell [1].

### writing cells — smooth SDF voxels

Smooth voxel terrain stores a <em>signed distance field (SDF)</em> — each cell holds a floating-point number whose sign tells you which side of the nearest surface you're on and whose magnitude tells you how far. The surface itself is the zero-crossing of that field.

Editing an SDF grid is still a write loop over a brush shape, but the write is a blend rather than a replace. To add matter (fill), you push the stored value toward a more-negative number (deeper inside). To remove matter (dig), you push it toward a more-positive number (deeper outside). The strength parameter controls how much you move it per frame, which gives you the smooth, clay-like feel of sculpting tools.

```python
# Dig a sphere out of SDF terrain
for each cell (x, y, z) within AABB of sphere:
    d = distance(cell_center, p) - radius   # +outside, -inside brush
    grid[x, y, z].sdf = max(grid[x,y,z].sdf, d)  # union of two SDFs = take max
```

This is <em>constructive solid geometry (CSG)</em> on the grid — the same union, subtraction, and intersection operations that CAD systems use on analytic geometry, applied cell-by-cell to a discrete SDF [2]. Union takes the minimum of two SDF values (keep the closer surface). Subtraction takes the maximum of the original field and the negated brush field (carve away). Smooth blending replaces the sharp min/max with a soft approximation, giving rounded edges between operations. See [sdf and csg modeling](../generating/sdf-and-csg-modeling.md) for how these same operations are used at generation time — runtime editing reuses exactly the same math, just applied one brush stroke at a time.

An important property: SDF values are clamped, so changes stay local. If you dig a hole at ground level, only nearby cells need to change; cells far from any surface are already saturated at their maximum clamped value and don't need updating [1].

---

## the efficiency rule — mark dirty, re-mesh only what changed

### chunks and dirty bits

A naive implementation re-meshes the entire world after every edit. A real one re-meshes only the smallest set of chunks that the edit actually touched.

Every chunk carries a <em>dirty flag</em> — a single bit that says "my mesh is out of date and needs rebuilding." When an edit writes new cell values, the engine walks the set of changed cells, finds which chunk each cell belongs to, and sets that chunk's dirty flag [1].

```
edit writes cells → collect affected (chunk_x, chunk_y, chunk_z) set
for each unique chunk in that set:
    chunk.dirty = true
    enqueue chunk in rebuild_queue
```

On the worker thread side, the rebuild queue drains continuously: pick a dirty chunk, run the mesher on its current data, upload the new mesh, clear the dirty flag. The [threading and meshing pipeline](./threading-and-meshing-pipeline.md) covers how that queue is scheduled and paced — here the key point is that the dirty flag is the handoff between the edit path and the mesh path.

### neighbor propagation

An edit near the edge of a chunk doesn't just affect that chunk's mesh. Because meshing any chunk requires reading voxel data from all six face-neighbors (and potentially all twenty-six neighbors, including diagonals, for ambient occlusion — more on that below), cells changed near an edge may change what the *neighboring* chunk's mesher sees.

The rule: mark a chunk dirty if the edit touched any cell within the chunk *or* within one cell of the chunk's boundary on any axis.

```
for each changed cell:
    mark chunk(cell) dirty
    if cell.x == chunk.min_x:   mark chunk(cell).x_minus_neighbor dirty
    if cell.x == chunk.max_x:   mark chunk(cell).x_plus_neighbor dirty
    # repeat for y, z faces
```

For ambient occlusion with all-26-neighbor dependency, a corner edit can touch up to seven neighboring chunks (three face-neighbors, three edge-neighbors, one corner-neighbor) [3]. Most engines mark all six face-neighbors as a conservative safe default rather than computing the minimal affected set — the overhead of a few extra re-meshes is cheaper than the complexity of tracking diagonal dependencies precisely.

The [chunk management and streaming](./chunk-management-and-streaming.md) page covers how the engine knows which chunks are resident and addressable at any moment.

---

## boundary correctness — seams at chunk edges

### the padding trick

When the mesher runs on a chunk, it needs voxel data for cells that may belong to neighboring chunks — to decide whether a face is visible, and to compute ambient occlusion at boundary vertices. The standard solution is to give each chunk a one-voxel-wide padding border: an 18×18×18 working buffer for a 16³ chunk, where the six outer faces are filled with data copied from the neighbor chunks before the mesher starts [3].

The mesher only produces geometry for the inner 16³ cells, but it reads from all 18³ to make correct decisions at the boundary. This means: before you mesh a chunk, you must have current data from all six neighbors. If a neighbor is mid-edit and its data is stale, you hold off or accept a one-frame seam.

### AO seams

Ambient occlusion (AO) is the most demanding of the cross-chunk data requirements. Computing the AO value at a vertex on the edge of chunk A requires knowing whether the neighboring cells in chunk B are occupied — and the same vertex may appear in both chunk A's mesh and chunk B's mesh, and must show the same AO value in both or a visible seam appears.

The AO formula for a voxel face vertex depends on three adjacent cells (two side-neighbors and one corner-neighbor) [4]:

```
vertexAO(side1, side2, corner):
    if side1 AND side2: return 0       # fully occluded
    return 3 - (side1 + side2 + corner)
```

When a vertex sits at a chunk boundary, some of those three adjacent cells are in a different chunk. Both chunk meshers need the same snapshot of the neighbor's data when they compute that vertex. This is why dirty-marking propagates to neighbors: chunk B's dirty flag ensures it re-meshes using chunk A's updated data after an edit, so both sides of the boundary converge on the same AO values.

There is also a quad-orientation correctness issue: greedy meshing merges adjacent faces into larger quads. For a merged quad to shade correctly with per-vertex AO, all four corner AO values must be consistent — otherwise you must flip the quad diagonal so interpolation goes the right way [4]. An edit that changes the AO value at one corner of a merged quad forces that quad to be split or re-oriented, which is one reason why a single edited cell can ripple outward to affect a larger meshing region than you might expect.

Lighting (light propagation from voxel sources and sky) has similar cross-chunk dependencies. See [light propagation](../simulation/light-propagation.md) for how the lighting system's dirty-and-recompute cycle meshes with the editing cycle.

---

## undo and redo — edit deltas, not snapshots

### why store deltas

After an edit you need to be able to undo it. The naive approach: snapshot the entire affected region of voxel data before each edit, store the snapshots in a stack, pop and restore on undo.

The problem is size. A brush edit that touches a 10×10×10 volume stores a 1000-cell snapshot for every single brush stroke. A large sculpting session generates gigabytes of snapshot data in minutes.

The better approach: store only the <em>edit delta</em> — the set of (cell position, old value, new value) pairs for the cells the edit actually changed. Undo replays the old values; redo replays the new values. A delta for a sphere brush that changed 500 cells stores 500 records regardless of brush radius, not 500³.

```
# Before edit
delta = []
for each cell about to change:
    delta.append((pos, old_value=grid[pos], new_value=target))

# Apply
for (pos, old, new) in delta:
    grid[pos] = new

# Undo
for (pos, old, new) in reversed(delta):
    grid[pos] = old
```

WorldEdit — the canonical Minecraft world-editing tool — stores only the previous state of changed blocks per operation, up to a configurable session limit (default 15 operations) [5]. The pattern is the same across every voxel system that supports undo.

### command objects vs raw deltas

A cleaner abstraction wraps the delta in a *command object* that knows both how to apply itself and how to reverse itself. This is the Command pattern applied to voxel edits. Commands compose: a "paste structure" command can be one entry in the history stack even though it touched thousands of cells. Commands also serialize cleanly for sending over the network.

---

## multiplayer — ordering and conflict

### send commands, not voxel data

Sending raw voxel data over a network is expensive. A single edited chunk at 16³ × 2 bytes = 8 KB per chunk per change; a busy multiplayer session generates far too much data. The alternative that real systems use: send the *edit command* (brush type, position, radius, material) and let each client re-execute the command on its own copy of the world data.

Teardown uses exactly this pattern: the server sends destruction commands to all clients, and each client applies them identically and in the same order. The constraint is determinism — every client must apply commands in the same sequence to stay synchronized [6].

### ordering and "last write wins"

When two players edit overlapping regions simultaneously, their commands arrive at each client in potentially different orders. The simplest resolution strategy is <em>last-write-wins</em> at the cell level: whichever command writes a cell last (by server timestamp or sequence number) wins. This is safe for cell-granular data because each cell is independent — there is no structural dependency that a conflicting write could violate.

For undo in a multiplayer context the problem deepens. If player A executes undo, they intend to undo *their own last action*, not player B's last action. Systems that support per-player undo in multiplayer need to track a per-player delta stack and apply undo as a filtered command that only reverses cells that player's command changed — leaving cells changed by other players untouched.

The general collaborative-editing literature offers two frameworks for this problem. Operational Transformation (OT) rewrites incoming operations to account for concurrent edits that have already been applied. Conflict-free Replicated Data Types (CRDTs) structure the data so that any merge order produces the same result. For voxel data specifically, a Last-Write-Wins Register CRDT per cell (with server-assigned sequence numbers) is the most practical: each cell is an independent register, concurrent writes converge deterministically to whichever has the higher sequence number, and the implementation is trivially parallel [7].

Teardown notes a practical constraint on command-based sync: late-joining players must receive the entire command history to reconstruct current world state. Once accumulated destruction is large, the join delay becomes prohibitive, and the game simply disables mid-session joins above a history size threshold [6].

---

## cost of editing different stores

Not all voxel storage structures pay the same price for runtime writes. This matters because your choice of storage structure (covered in depth at [choosing a voxel store](../storing/choosing-a-voxel-store.md)) directly controls how expensive every user edit is.

| store | read cost | write cost | why |
|---|---|---|---|
| flat dense array | O(1) — index math | O(1) — index math | direct address, no traversal |
| chunked flat arrays | O(1) within chunk | O(1) + dirty flag | same as flat, plus chunk lookup |
| octree | O(log n) | O(log n) + subtree update | must walk pointers to leaf, may need node splits |
| sparse voxel DAG (SVDAG) | O(log n) | very expensive | shared nodes — one edit may invalidate thousands of deduplicated references |

**Flat arrays are the easiest store to edit.** A write is an array index computation — three multiplies and two adds — then a memory write. No traversal, no pointer updates [8].

**Octrees are significantly slower to edit.** Each edit requires walking the tree from root to the target leaf, potentially splitting nodes along the way if the target cell was previously represented by a coarser node. Frequent small edits cause thrashing as the tree repeatedly splits and could merge [8].

**SVDAGs and DAG-compressed structures are the hardest to edit at runtime.** The whole point of a DAG is that identical subtrees share a single node in memory. One edited cell potentially invalidates the shared reference for every other subtree that was deduplicating against that node. The write cannot stay local — it must either break the sharing (increasing memory back toward octree size) or rebuild the deduplication structure, which is expensive. SVDAGs are excellent for static or slowly-changing content; they are a poor fit for per-frame player edits [8][9].

This is the core tradeoff: the structures that compress static voxel data most aggressively are exactly the structures that make runtime editing most expensive. Systems that need both (e.g. large static environments with player-editable zones) often use a hybrid: compress the static background in a DAG, and maintain a separate flat-array edit layer that records the delta between the DAG state and current state.

---

## putting it together — the edit cycle

One brush stroke in a voxel editor or game traverses this sequence:

1. **Input** — user action defines a brush (shape, position, radius, material/mode).
2. **Cell write** — write loop iterates cells in the brush shape; for blocky voxels sets material ID; for SDF voxels blends toward the brush SDF using min/max (CSG union/subtract).
3. **Delta record** — capture (pos, old, new) for every changed cell into the current undo command.
4. **Dirty marking** — for every changed cell, mark its chunk dirty; for any changed cell within one voxel of a chunk boundary, mark that neighbor dirty too.
5. **Rebuild queue** — dirty chunks enter the worker queue; the mesher runs on each, reading the updated voxel data plus a one-voxel padding border from each neighbor.
6. **Mesh upload** — completed mesh data is posted back to the main thread; the GPU buffer is updated.
7. **Network** — the edit command is sent to the server/peers; remote clients execute the same brush logic on their own copy of the data.
8. **Undo stack** — the delta command is pushed onto the per-player history stack.

The whole cycle from brush input to visible mesh update takes one to several frames depending on brush size and worker thread load. Steps 5–6 run asynchronously in a well-implemented engine, so the main thread is never blocked — the user sees the old mesh for one frame and the new mesh the next.

---

## references

[1] Zylann. *Godot Voxel Tools — Performance*. Read the Docs. [source](https://voxel-tools.readthedocs.io/en/latest/performance/) (Retrieved June 2026). (Production voxel engine documentation covering dirty chunk tracking, spatial locking, bulk operations, and time budgets.)

[2] Wegen, O., Döllner, J., and Trapp, M. (2022). "Interactive Editing of Voxel-Based Signed Distance Fields." *Journal of WSCG*, 30. DOI: 10.24132/JWSCG.2022.9. [local PDF](../papers/wegen-2022-interactive-editing-voxel-sdf.pdf) · [source](https://dspace5.zcu.cz/handle/11025/49396) (Presents GPU-accelerated interactive CSG editing of SDF voxel grids including copy, move, union, and subtraction operators.)

[3] Spacefarer dev team. *Voxel Meshing for the Rest of Us*. Spacefarer devblog. [source](https://playspacefarer.com/voxel-meshing/) (Retrieved June 2026). (Details the 18×18×18 padding approach for chunk-boundary correct meshing.)

[4] Bloom, M. (0fps.net). "Ambient Occlusion for Minecraft-like Worlds." 2013. [source](https://0fps.net/2013/07/03/ambient-occlusion-for-minecraft-like-worlds/) (Retrieved June 2026). (Derives the vertex AO formula and the quad-flip rule; explains why AO requires all 26 neighbors.)

[5] EngineHub. *WorldEdit — History*. WorldEdit 7.4 documentation. [source](https://worldedit.enginehub.org/en/latest/usage/general/history/) (Retrieved June 2026). (Describes the delta-based undo stack: only changed blocks are stored, up to a session limit.)

[6] Gustafsson, D. (Tuxedo Labs). "Teardown Developer Breaks Down Multiplayer and Voxel Destruction Tech." 80.lv. [source](https://80.lv/articles/teardown-developer-breaks-down-multiplayer-and-voxel-destruction-tech) (Retrieved June 2026). (Explains the command-based synchronization strategy, determinism requirement, and join-in-progress history limit.)

[7] Shapiro, M., Preguiça, N., Baquero, C., and Zawirski, M. (2011). "Conflict-free Replicated Data Types." *Proceedings of SSS 2011*, LNCS 6976, 386–400. arXiv:1806.10254. [source](https://arxiv.org/pdf/1806.10254) (Retrieved June 2026). (Defines CRDT semantics; Last-Write-Wins Register maps directly to per-cell voxel conflict resolution.)

[8] bink.eu.org. *Fast Voxel Data Structures*. [source](https://bink.eu.org/fast-voxel-datastructures/) (Retrieved June 2026). (Benchmarks and analysis of flat array, grid hierarchy, octree, and DAG for read and write operations.)

[9] Kämpe, V., Sintorn, E., and Assarsson, U. (2013). "High Resolution Sparse Voxel DAGs." *ACM Transactions on Graphics*, 32(4), Article 101. DOI: 10.1145/2461912.2462024. [local PDF](../papers/kampe-sintorn-assarsson-2013-high-resolution-sparse-voxel-dags.pdf) · [source](https://dl.acm.org/doi/10.1145/2461912.2462024) (The canonical SVDAG paper; the deduplication mechanism that makes DAGs memory-efficient also explains why per-cell writes are structurally expensive.)
