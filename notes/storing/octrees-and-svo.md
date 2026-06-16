<link rel="stylesheet" href="./css/globals.css">

# octrees and sparse voxel octrees

You want to store a huge 3D world — a city block, a mountain range, a detailed character — at voxel resolution. The voxel count explodes as a cube of the resolution: a 1024³ grid holds a billion cells. But almost all of them are empty air or solid rock with no interesting variation. The goal is to store only the parts that matter, get the rest for free, and — as a bonus — have a built-in level-of-detail system that costs nothing extra.

The hierarchical structure that achieves this is the <em>octree</em>. Its sparse variant, the <em>sparse voxel octree (SVO)</em>, is the canonical tool for large, mostly-empty volumes and for voxel ray tracing. Understanding it unlocks both compact storage and LOD-for-free rendering.

The starting point for why any of this is needed is [the storage problem](./the-storage-problem.md) and how [dense grids and chunks](./dense-grids-and-chunks.md) fall short at scale.

---

## the coarse mental model

Take the bounding box of your volume. Split it in half along each axis — X, Y, and Z — and you get 8 equal child boxes. Each child can be split the same way, producing 8 grandchildren. Keep going until the boxes are voxel-sized.

At every level, if a box is **entirely empty**, stop — don't subdivide it further and don't store its children. If a box is **entirely full or uniform**, stop — record the value and you're done. Only **mixed** boxes need to be split further.

The result is a tree where each node has up to 8 children. The vast majority of nodes — the ones covering empty space — simply don't exist. You only pay for the structure around the surface of your geometry and any internal variation. This is the octree.

A beginner can hold that model and stop here. The rest of this page peels it open.

---

## recursive subdivision — how the tree is built

Start with a single cube at depth 0 that covers the entire volume. Each subdivision step splits a cube at depth d into 8 children at depth d+1, cutting the edge length in half along each axis. The 8 children are the 8 octants of the parent cube.

```
depth 0:   one cube — the whole volume

depth 1:   8 cubes, each 1/8 the volume

depth 2:   up to 64 cubes — but most are empty and discarded

depth N:   leaf voxels — cubes at grid resolution
```

The splitting rule at each node:
- **empty octant** — no children; the node is not stored at all
- **uniform octant** — leaf node; store its value once
- **mixed octant** — internal node; create up to 8 children and recurse

A node's depth tells you the voxel size it represents: a node at depth d covers `(world_size / 2^d)` units on a side. This fact will matter a lot when we get to LOD.

The internal node is more than a branching point — it is a <em>coarse summary of its subtree</em>. When an SVO stores appearance data (color, surface normal, opacity) at each internal node as the average of its children, that node becomes a valid coarser representation of everything below it.

---

## sparsity: the key idea

A dense grid at 1024³ allocates 2^30 ≈ 1 billion cells unconditionally. An octree doesn't. It allocates a node only when a region is mixed — only at the surface of geometry and at boundaries between materials.

Take a scene that is 99% empty air (common in architectural or game worlds). A dense 1024³ grid wastes memory on nearly all of it. An octree at the same resolution allocates nodes only around the solid geometry. The tree depth is log₂(1024) = 10 levels, and only the O(surface_area) nodes near the boundary are ever created.

This is the <em>sparse voxel octree</em>: an octree applied to voxel data where the primary resource being conserved is the empty interior. Laine & Karras [1] made this structure practical for GPU ray tracing, and GigaVoxels [2] showed it could stream billions of voxels in real time.

---

## node layout — how the tree is actually stored

How you store the tree matters as much as what the tree represents. Two main approaches exist.

### pointer-based nodes

The most direct layout stores, per node:
- a **valid mask** (8 bits): one bit per octant; bit i is set if child i is non-empty and exists
- a **leaf mask** (8 bits): one bit per octant; bit i is set if child i is a leaf (voxel data), not a subtree
- a **child pointer**: an offset to the block of children in memory

The key insight from Laine & Karras [1] is that children are stored **contiguously** — all children of a given node sit side by side in memory. A single child pointer is enough to reach all of them: to find child i, read the ValidMask, count the set bits below i (the popcount of the low bits), and offset from the child pointer by that count. This makes the per-node cost small: 8 + 8 bits of masks plus one pointer — roughly 8 bytes per node for the geometry descriptor.

A **far pointer** flag handles the uncommon case where children are placed far from the parent in memory and the short relative offset overflows.

Optionally each node carries **contour data**: a compact geometric approximation (e.g., a plane equation) that describes the local surface. During ray casting, if the contour matches the surface closely enough, the traversal stops at this level rather than descending further — trading geometric detail for a controlled approximation.

A small sketch of one internal node:

```
[ ValidMask: 8 bits ]  — which octants exist
[ LeafMask:  8 bits ]  — which of those are leaves
[ ChildPtr:  N bits ]  — offset to child block
[ Contour:  optional ] — surface approximation
```

### pointerless / linearized layouts

An alternative is to lay the tree out in breadth-first order in a flat array so that the children of node i are always at a calculated index — no pointers needed. This works cleanly for **complete** trees (all levels fully populated). For sparse trees you need either:

- a **locational code** (Morton code): encode the path from root to node as a sequence of 3-bit octant numbers; children are the parent's code with one octant appended; store in a hash map for O(1) lookup
- a **hash grid**: abandon the explicit tree and use the Morton code directly as a hash key

Pointerless layouts save the memory occupied by pointers and allow certain traversal algorithms to avoid following pointers at all. The tradeoff is that random access to an arbitrary node requires either a hash lookup or computing the array index, rather than dereferencing a stored pointer.

| layout | pointer overhead | random access | dynamic edits | suited for |
|---|---|---|---|---|
| pointer-based (block ptr) | 1 ptr per node | fast, one dereference | tree surgery, manageable | GPU ray casters, static/semi-static scenes |
| 8 ptrs per node | 8 ptrs per node | fast | easy per-child | small dynamic trees |
| BFS array (complete tree) | none | O(1) arithmetic | expensive rebalance | full, dense trees |
| locational code + hash map | none | O(1) hash | insert/delete at key | sparse, dynamic trees |

---

## LOD for free

Here is where the octree pays a structural dividend. Because each tree level represents a **coarser subdivision**, you already have a ready-made LOD hierarchy inside the tree.

A ray caster descending the tree does not have to go all the way to leaf depth. It stops when the projected voxel size at the current node is smaller than a pixel — meaning further subdivision would be sub-pixel and invisible. It reads the summary data stored at that internal node (the averaged color and normal from the subtree) and shades.

- A ray hitting a distant part of the scene stops early, at a coarse node. Fast, and the result looks right.
- A ray hitting a close surface descends to leaf depth. Accurate detail where the eye can see it.

No separate LOD pipeline. No manual LOD authoring. The tree depth at which traversal terminates is the LOD level for that ray, and it varies naturally per-pixel based on distance and angle. This is the mechanism used in GigaVoxels [2] and in SVO ray tracing [3] — see [sparse voxel octree ray tracing](../rendering/sparse-voxel-octree-raytracing.md) for the full rendering discussion, and [LOD in engines](../engines/lod-in-engines.md) for how this compares to mesh-based LOD pipelines.

GigaVoxels extended this idea into streaming: the ray caster marks which tree nodes it needs; the CPU uses those marks to stream only the required subtrees from disk into GPU memory. A scene of 8192³ voxels fits in limited GPU memory because only the visible, needed nodes are resident at any time [2].

---

## costs — what the octree gives up

The SVO is not a replacement for a flat array; it is a different tradeoff. Understanding the costs tells you when to reach for it and when not to.

### access is O(depth), not O(1)

Looking up the value at a world-space point requires traversing the tree from root to leaf: check the valid mask at each node, follow the child pointer, repeat. That is `log₂(resolution)` steps — ten steps for 1024³ — each potentially a cache miss if the tree is scattered in memory. A flat array lookup is a single multiply-add-index with no branching.

For workloads that need random access to arbitrary voxels — physics simulation sweeping over every cell, for example — a flat array or [hash grid](./hash-grids-and-bricks.md) often wins.

### runtime edits mean tree surgery

Adding or removing a voxel is not a simple array write. You must:
1. Traverse from root to the target leaf — O(depth) steps
2. Update the leaf (or split a uniform node to create the leaf)
3. Update the ValidMask and LeafMask at every ancestor along the path
4. Potentially merge siblings back into a uniform parent if the edit makes a subtree uniform

Each step is a dependent memory access. For workloads with frequent, scattered edits — destructible terrain, voxel painting — this surgery is expensive. At large scale, a practical answer is to combine many **smaller trees** at a top-level grid: a tree that covers a 16³ chunk, say, where edits to that chunk only touch that tree. This is the approach taken by [OpenVDB and NanoVDB](./openvdb-and-nanovdb.md).

### memory for dense scenes

For a scene that is mostly full — a solid rock with no interior void — the SVO overhead (one node per voxel plus ancestors) can exceed the cost of a flat array. SVOs win only when the scene is sparse enough that the unskipped empty space exceeds the tree overhead.

---

## the next step: SVDAGs

The SVO prunes empty space. But it does not prune **repeated geometry**. A checkerboard wall, a forest of identical trees, a tiled floor — each instance is its own subtree, stored separately, even if it is byte-for-byte identical to another.

Kämpe, Sintorn & Assarsson [4] showed that merging identical subtrees turns the tree into a directed acyclic graph — the <em>sparse voxel DAG (SVDAG)</em>. The reduction is dramatic: 1 to 3 orders of magnitude fewer nodes in all tested scenes, including highly irregular ones. The tradeoff is that the SVDAG needs explicit child pointers (not a single block pointer) because children are no longer stored contiguously after deduplication.

The full treatment is in [sparse voxel DAGs](./sparse-voxel-dags.md).

---

## when to use an SVO

**Reach for an SVO when:**
- the scene is large and mostly empty (air, outer space, sparse geometry)
- you want LOD for free with a ray casting renderer
- the data is read-mostly or static (built once, traversed many times)
- you are voxelizing high-resolution geometry that would be prohibitive as a dense grid

**Reach for something else when:**
- you need O(1) random access and dense uniform data → [dense grid or chunk store](./dense-grids-and-chunks.md)
- you need fast, frequent edits → hash grid or brick store — see [hash grids and bricks](./hash-grids-and-bricks.md) and [choosing a voxel store](./choosing-a-voxel-store.md)
- you need maximum compression and the geometry has repeated patterns → [SVDAG](./sparse-voxel-dags.md)

---

## references

[1] Laine, S. and Karras, T. (2010). "Efficient Sparse Voxel Octrees." *ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (I3D 2010)*. NVIDIA Research. [local PDF](../papers/laine-karras-2010-efficient-sparse-voxel-octrees.pdf) · [source](https://research.nvidia.com/sites/default/files/pubs/2010-02_Efficient-Sparse-Voxel/laine2010i3d_paper.pdf)

[2] Crassin, C., Neyret, F., Lefebvre, S., and Eisemann, E. (2009). "GigaVoxels: Ray-Guided Streaming for Efficient and Detailed Voxel Rendering." *ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (I3D 2009)*. [source](https://hal.science/inria-00345899)

[3] Crassin, C., Neyret, F., Sainz, M., Green, S., and Eisemann, E. (2011). "Interactive Indirect Illumination Using Voxel Cone Tracing." *Computer Graphics Forum*, 30(7), 1921–1930. [local PDF](../papers/crassin-2011-voxel-cone-tracing.pdf) · [source](https://research.nvidia.com/publication/2011-09_interactive-indirect-illumination-using-voxel-cone-tracing)

[4] Kämpe, V., Sintorn, E., and Assarsson, U. (2013). "High Resolution Sparse Voxel DAGs." *ACM Transactions on Graphics* (SIGGRAPH 2013), 32(4). DOI: 10.1145/2461912.2462024. [local PDF](../papers/kampe-sintorn-assarsson-2013-high-resolution-sparse-voxel-dags.pdf) · [source](https://dl.acm.org/doi/10.1145/2461912.2462024)

[5] Meagher, D. (1982). "Geometric modeling using octree encoding." *Computer Graphics and Image Processing*, 19(2), 129–147. DOI: 10.1016/0146-664X(82)90104-6. (Paywalled — the founding paper on octree encoding for geometric solids.)
