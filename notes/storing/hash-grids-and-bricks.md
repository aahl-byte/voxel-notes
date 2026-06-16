<link rel="stylesheet" href="./css/globals.css">

# hash grids and brick maps

You are building a room-scale 3D scanner that fuses a thousand depth frames per minute into a single live surface. Or a voxel sculpting tool where every brush stroke carves, fills, and rewrites thousands of cells. Or a global-illumination system that needs to sample a sparse light-cache volume from a GPU shader, 60 times a second.

All three need the same thing: a sparse volume you can **read in constant time and edit in constant time**, without rebuilding any tree. A fully dense grid — say 512³ at 4 bytes per voxel — would cost 512 MB for a box roughly ten meters across at centimetre resolution. Most of that box is air. What you actually want is a structure that stores only the occupied cells, reaches any one of them in a single flat lookup, and can insert or erase a cell the instant a depth frame arrives or a brush stroke lands. That is what this page covers.

See [the storage problem](./the-storage-problem.md) for why a dense grid overflows memory, and [octrees and SVOs](./octrees-and-svo.md) for the tree-based alternative these structures deliberately avoid.

---

## the coarse model — two flat sparse structures

There are two ideas here, and they are related:

**Spatial hashing** stores each occupied cell directly in a hash table, keyed by its integer (x, y, z) coordinate. There is no grid allocated for empty cells; only the occupied ones exist in memory. Lookup is a single hash computation plus one table probe — O(1) expected, no depth, no traversal.

**Brick maps** (also called tile grids) store occupied cells in small dense blocks — "bricks" of perhaps 8×8×8 voxels — and keep a sparse index of those bricks. Cells inside a brick are addressed by a fixed-size array offset (dense, fast, cache-friendly); the index itself is sparse and is often a hash table. You get the O(1) index access of hashing AND the locality and GPU friendliness of a packed 3D array.

The typical story in practice: spatial hashing wins for maximum flexibility and simplest editing; brick maps win when the access pattern benefits from spatial locality — which is almost all rendering and reconstruction workloads. [VDB and NanoVDB](./openvdb-and-nanovdb.md) formalize the brick-map idea into a production data structure with an extra layer of internal nodes.

---

## spatial hashing — storing occupied cells in a table

### the core idea

Take every occupied voxel at integer coordinate (x, y, z) and map it to a slot in a flat array (the hash table) using a hash function. The hash function turns the coordinate into an index. Cells that hash to the same index are **collisions** — handled by linear probing (try the next slot) or by fixed-size buckets (each table entry holds a small list).

The hash function most widely used for 3D integer coordinates comes from Teschner et al. [1]:

```
h(x, y, z) = (x · 73856093 ⊕ y · 19349663 ⊕ z · 83492791) mod n
```

Multiply each coordinate by a distinct large prime, XOR the three results, take modulo the table size `n`. The primes are chosen so that nearby coordinates produce scattered table indices — minimising clustering — while the XOR keeps the function cheap to evaluate on both CPU and GPU.

**Why this beats a tree for editing:**
- Insert a new cell: compute the hash, find a free slot, write. One flat array write.
- Erase a cell: find the slot, mark it free. One flat array write.
- No parent pointers to update. No rebalancing. No node splitting.

Contrast with an [octree](./octrees-and-svo.md): inserting a cell that falls below a partially-empty node may require allocating new child nodes, updating parent pointers, and potentially re-merging on erase. Every edit is surgery on a tree whose depth is proportional to log₂(resolution).

### the collision budget — load factor

The fraction of table slots that are occupied is the <em>load factor</em>. A table at 50% load (half its slots filled) is fast: most lookups find their cell in one probe. A table at 90% load degrades badly — many probes per lookup, poor cache behaviour. Real systems aim for 25–50% load, which means intentionally allocating roughly twice the memory needed for the actual data. That waste buys speed.

Niessner et al. [4] implement this with a bucket structure: each hash index addresses a fixed-size bucket of 10 entries (`HASH_BUCKET_SIZE = 10`), and overflow beyond those 10 uses a linked-list extension. Each voxel block is 8×8×8 (`SDF_BLOCK_SIZE = 8`). Their GPU CUDA kernels insert using `atomicCAS` — compare-and-swap — so thousands of depth pixels can allocate new blocks in parallel without data races:

```c
// atomic: transition FREE_ENTRY → LOCK_ENTRY, then write
prevWeight = atomicCAS(&d_hash[i].ptr, FREE_ENTRY, LOCK_ENTRY);
if (prevWeight == FREE_ENTRY) {
    d_hash[i] = entry;  // we own this slot
    return true;
}
```

The GPU builds or updates the entire hash table each frame by launching one CUDA thread per incoming depth pixel — fully parallel, no serial bottleneck.

### perfect spatial hashing — eliminating collisions entirely

For **static** data (a fixed sparse set that never changes), Lefebvre and Hoppe [2] show how to precompute a two-level hash function with **zero collisions**. Every lookup is exactly two memory reads: one into a small offset table, one into the data table. The offset table is precomputed offline to route every key to a unique slot. The result is ideal for read-only sparse volumes baked at asset-build time — geometry caches, voxelised scene data, sparse textures — because the two-memory-access bound is as fast as any hash scheme can be, and it preserves spatial coherence so nearby coordinates tend to land in nearby memory addresses (better cache hit rates than a pseudorandom hash).

The trade-off: precomputation is expensive and must be re-run if even a single cell changes. **Use perfect spatial hashing for read-only data; use standard open-addressing hashing for live editing.**

### when spatial hashing wins

- Live reconstruction — new cells arrive every frame from a sensor, positions unknown in advance.
- Physics simulation — particles or rigid bodies occupy unpredictable, changing regions.
- Sparse collision detection — Teschner et al. [1] used it specifically for this.
- Any workload where simplicity matters and spatial locality of access is low.

---

## brick maps — sparse index over dense blocks

### the core idea

Instead of hashing individual voxels, divide space into a regular grid of fixed-size **bricks** — say 8×8×8 or 32×32×32 voxels — and store only the bricks that contain at least one occupied cell. An index maps each brick's grid coordinate to where that brick's data lives in memory. Inside a brick, every voxel slot is allocated whether or not it is occupied — the brick is a plain dense 3D array.

This is a hybrid: sparsity between bricks, density within them.

```
world coordinate (wx, wy, wz)
  → brick index  (bx, by, bz) = (wx >> 3, wy >> 3, wz >> 3)  [for 8^3 bricks]
  → local offset (lx, ly, lz) = (wx & 7,  wy & 7,  wz & 7)
  → data address = index[bx, by, bz] + (lz * 8 + ly) * 8 + lx
```

Two lookups: one into the sparse index (which may be a hash table or a fixed array), one into the brick's dense array. No tree, no traversal.

The key term for this pattern is <em>brick map</em> (also called a tile grid or paged volume). GigaVoxels [3] used 32³ bricks; Niessner et al. [4] and most real-time reconstruction systems use 8³ blocks. See [memory layout and Morton codes](../optimization/memory-layout-and-morton.md) for how brick contents are often stored in Morton order to improve 3D cache locality within the brick.

### why the block size matters

The brick is the granularity of both allocation and access:

- **Too small** (e.g., 2³ = 8 voxels): the index overhead dominates; most of the memory is index entries, not voxel data. Editing at voxel granularity is fast, but memory is wasted on pointers.
- **Too large** (e.g., 64³ = 262144 voxels): each brick wastes most of its 3D array on empty interior cells; the structure is nearly as wasteful as a dense grid for thin surfaces.
- **8³ to 16³ is the practical sweet spot** for real-time workloads: fits in L1/L2 cache, matches the grain of GPU memory transactions, and keeps the index small enough to hash cheaply.

### brick maps on the GPU

A brick of 8×8×8 voxels stored contiguously is 512 values. A single warp (32 GPU threads) can load the entire brick in a small number of coalesced memory transactions — **coalesced** because the values are contiguous in memory, which is the GPU's requirement for efficient access. Sparse hashing of individual voxels does not give this: nearby voxels are scattered across the hash table, so a warp reading a spatial neighborhood makes dozens of non-coalesced accesses.

GigaVoxels [3] stores bricks in a 3D GPU texture array (the brick pool). Texture memory has hardware-managed 2D/3D spatial caching, so reading a brick that fits in the texture cache has essentially free access to all 512 voxels. The sparse octree node above each brick stores a pointer (texture offset) into this pool. When a node's brick is evicted due to memory pressure, the pointer is invalidated and the brick is re-requested asynchronously — the GPU's brick cache implements demand paging.

### when to use a brick map instead of plain spatial hashing

| situation | prefer |
|---|---|
| mostly reading, locality matters (rendering, cone tracing) | brick map |
| GPU shader needs predictable memory access | brick map |
| edits arrive at arbitrary positions every frame | either; brick map slightly more overhead per edit |
| data is fully dynamic, positions completely unpredictable | spatial hash |
| static or infrequently updated volume | brick map or perfect spatial hash |
| need implicit LOD (coarse bricks at top of hierarchy) | brick map with hierarchy (→ VDB) |

---

## why both lose to trees for some things

Flat hash structures have a real weakness: **no implicit level of detail**. An octree naturally encodes LOD — the coarser levels are already there as inner nodes with averaged values. A hash grid or brick map stores only the full-resolution data; any coarser representation must be built separately and stored in additional tables. This makes hash structures less natural for:

- Ray marching with adaptive step sizes (an octree can jump over empty space using its inner-node "is this subtree empty?" bit).
- Distant-LOD rendering where you want a coarser version of the volume without ray-marching through full resolution.
- Memory-hierarchy traversal where the coarse levels fit in cache but fine levels do not.

The trade-off in plain language: **trees buy LOD and empty-space skipping for free; flat structures buy edit speed and constant-depth access for free.** [Choosing a voxel store](./choosing-a-voxel-store.md) works through this comparison end to end.

---

## what VDB formalizes — and what GPUs take from it

VDB (see [OpenVDB and NanoVDB](./openvdb-and-nanovdb.md)) is, at its core, a multi-level brick map with a B+tree-like fixed-depth index. The root node is a hash table of large regions. Below it are one or two levels of internal nodes (dense arrays, not hash tables). At the bottom are leaf nodes that are exactly dense 8×8×8 (or 4×4×4) voxel bricks. The fixed tree depth means every lookup is still a constant number of steps — VDB does not lose the O(1) character of brick maps, but it adds the implicit LOD and empty-space structure that a flat hash map lacks.

On the GPU, [NanoVDB and GPU voxel techniques](../optimization/gpu-voxel-techniques.md) flatten the VDB tree into a single memory buffer to eliminate pointer chasing — recovering the flat-access performance of a brick map while keeping VDB's hierarchical structure for ray traversal.

The practical summary: spatial hashing and brick maps are the vocabulary; VDB and NanoVDB are what production systems build by combining them with a shallow hierarchy.

---

## references

[1] Teschner, M., Heidelberger, B., Müller, M., Pomerantes, D., and Gross, M. H. (2003). "Optimized Spatial Hashing for Collision Detection of Deformable Objects." *Proceedings of Vision, Modeling, Visualization (VMV)*, 47–54. [local PDF](../papers/teschner-2003-optimized-spatial-hashing-collision-detection.pdf) · [source](https://matthias-research.github.io/pages/publications/tetraederCollision.pdf)

[2] Lefebvre, S. and Hoppe, H. (2006). "Perfect Spatial Hashing." *ACM Transactions on Graphics (SIGGRAPH)*, 25(3), 579–588. DOI: 10.1145/1179352.1141926. [local PDF](../papers/lefebvre-hoppe-2006-perfect-spatial-hashing.pdf) · [source](https://hhoppe.com/perfecthash.pdf)

[3] Crassin, C., Neyret, F., Lefebvre, S., and Eisemann, E. (2009). "GigaVoxels: Ray-Guided Streaming for Efficient and Detailed Voxel Rendering." *Proceedings of ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (I3D)*, 15–22. DOI: 10.1145/1507149.1507152. [local PDF](../papers/crassin-2009-gigavoxels-ray-guided-streaming.pdf) · [source](https://maverick.inria.fr/Publications/2009/CNLE09/CNLE09.pdf)

[4] Nießner, M., Zollhöfer, M., Izadi, S., and Stamminger, M. (2013). "Real-Time 3D Reconstruction at Scale using Voxel Hashing." *ACM Transactions on Graphics (SIGGRAPH Asia)*, 32(6), Article 169. DOI: 10.1145/2508363.2508374. [local PDF](../papers/niessner-2013-real-time-3d-reconstruction-voxel-hashing.pdf) · [source](https://niessnerlab.org/papers/2013/4hashing/niessner2013hashing.pdf)
