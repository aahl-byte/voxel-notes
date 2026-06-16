<link rel="stylesheet" href="./css/globals.css">

# sparse voxel octree ray tracing

Suppose you want to render a cathedral at 5mm geometric detail — every chisel mark on every stone, every edge sharp — across the entire building interior. No mesh budget on earth covers that. The triangle count would be in the billions. But if the geometry lives in a [sparse voxel octree](../storing/octrees-and-svo.md), you can cast rays directly through the hierarchy and produce a fully ray-traced image without ever converting to triangles. The tree itself becomes the render structure.

That is the goal of this page: understand how rays walk a voxel hierarchy to find the nearest surface, what makes the walk fast, and when this approach wins or loses against simpler alternatives.

---

## the coarse picture

Ray tracing a scene normally means: for each pixel, shoot a ray, find what it hits first, shade it. The expensive part is finding the hit. Over a flat list of geometry, that requires checking everything. A spatial hierarchy collapses that into a search — at each node, discard the subtrees the ray misses and descend only into the ones it enters.

A sparse voxel octree is already such a hierarchy (see [octrees and SVOs](../storing/octrees-and-svo.md)). Each node covers a cube of space and has up to eight children, each covering an eighth of that cube. Empty regions of space simply have no children — they are absent from the tree entirely. The key insight is that you can ray-trace this structure directly, without any separate acceleration structure: the octree IS the BVH.

The traversal works like this. Start at the root — a cube bounding the entire scene. The ray intersects it, so you look at the root's eight children and find which ones the ray enters. Descend into the nearest hit child. Repeat: look at its eight children, descend into the nearest hit child of those. When the ray exits a node without finding a filled child, pop back up to the parent and move on to the next sibling. Continue until you reach a leaf (a filled voxel) — that is your hit point — or the ray exits the tree entirely.

Three things make this fast:
- **empty space skips for free** — an absent subtree is skipped in one test, not thousands of DDA steps
- **free LOD** — stop descending when a voxel projects to one pixel; render at whatever resolution the view demands
- **geometry encoded in the hierarchy** — contour planes stored per voxel tighten the bounding volume, letting the traversal terminate earlier

This combination — walk the octree, push on hits, pop on misses, skip empty nodes wholesale — is <em>efficient sparse voxel octree (ESVO) ray casting</em>, formalized by Laine and Karras [1].

---

## the traversal in detail

### the three operations: push, advance, pop

The traversal loop maintains a current node, a stack of ancestors, and the ray's active t-interval (the range of distances along the ray still under consideration). At every step, exactly one of three things happens [1]:

- **PUSH** — the ray enters a child of the current node. Evaluate all eight children for ray intersection, find the nearest hit, save the current node on the stack, descend.
- **ADVANCE** — the ray passes through the current node without hitting any filled child. Move to the next sibling at this level.
- **POP** — the ray has exhausted all siblings at this level. Ascend to the parent (restored from the stack) and advance past it.

A small stack — one entry per octree level — is all the state required. The depth of an octree with resolution 2^L is at most L levels, so a 32K³ scene needs a stack of 15 entries.

Here is a minimal sketch of the idea in pseudocode:

```glsl
// stack-based SVO traversal — concept sketch, not the full ESVO algorithm
while not terminated:
    tc = ray_vs_cube(pos, scale)        // t-interval for current node

    if node_exists and t_min <= t_max:
        if voxel_is_small_enough():     // one voxel ≈ one pixel → leaf hit
            return t_min               // found the intersection
        // PUSH: descend into nearest hit child
        stack[scale] = (parent, t_max)
        idx  = nearest_child(tc, ray)
        pos, scale = child_cube(pos, scale, idx)
    else:
        // ADVANCE to next sibling, then POP if none remain
        pos, idx = step_along_ray(pos, scale, ray)
        if idx disagrees with ray direction:
            scale, parent, t_max = stack[highest_differing_level(pos)]
            pos = round_to_level(pos, scale)
```

The real ESVO algorithm encodes position in fixed-point bit strings so that the child slot index can be read directly from the bit-triplet at the current scale level — no division, no floating-point search [1].

### contours tighten the geometry

With axis-aligned cubes alone, a voxel occupying a cell that only partly intersects the real surface still looks like a full cube to the ray. This causes the silhouette to step-stair around smooth geometry.

Laine and Karras add a pair of oriented parallel planes — a <em>contour</em> — to each voxel, fitting the planes to the local surface orientation [1]. The traversal intersects the ray against the contour as well as the cube. If the ray misses the contour, the voxel is treated as a miss even though it hit the cube. This tightens the effective bounding volume of every voxel without subdividing further, allowing the traversal to prune more aggressively and producing smooth silhouettes at lower resolution.

Contours from ancestor nodes also contribute: the final shape of a voxel is the intersection of its own cube with all the ancestor contours it inherits. This gives several hierarchy levels' worth of geometric resolution improvement essentially for free.

### free level of detail

Because the traversal stops descending when a node is "small enough" — specifically when a voxel projects to approximately one pixel on screen — it naturally delivers level-of-detail without a separate LOD system. Near geometry gets full-depth traversal; distant geometry is resolved at a coarser level of the tree. The same octree handles all distances. This is the tree's LOD for free, without any mipmap pyramid or separate LOD asset.

---

## what makes it fast in practice

### empty space pruning

The biggest win over a uniform grid is empty-space skipping. In a [grid DDA traversal](./grid-ray-traversal.md), a ray through empty space steps one voxel at a time. In a sparse octree, an empty region the size of a subtree is skipped in a single intersection test against the parent node. A vast outdoor scene with a building in the center is almost entirely empty space at the top levels of the tree — those levels resolve to "miss" in a handful of tests.

### beam optimization

For primary rays (camera to scene), a useful pre-pass renders a coarse conservative depth image using the octree itself. The image is divided into 8×8 pixel blocks, and for each block, the traversal starts not at the root but at the first depth guaranteed to be closer than any geometry the block's rays can see. This skips the empty space above the scene for every ray, cutting iteration counts substantially — Laine and Karras report the effect visible as a reduction in traversal iterations across the image [1].

The beam optimization works specifically because of the voxel guarantee: if a node covers at least one ray in the block, it is not discarded. This guarantee fails for mesh-based acceleration structures (a coarse BVH might miss a thin feature), but holds for voxels because the voxel at that location truly bounds the geometry.

---

## tracing a DAG instead of a tree

For enormous static scenes, even a sparse octree can be too large. An SVO of a complex scene at 32K³ resolution may need gigabytes of GPU memory. The key observation: many regions of a real scene are geometrically identical — repeated bricks, tiles, foliage patterns, floor textures. An SVO stores duplicate subtrees for each instance.

A <em>directed acyclic graph (DAG)</em> merges identical subtrees into a single shared node, with multiple parent nodes all pointing to it. The geometry is described by which paths you take through the graph, not by which nodes exist — so merging identical subtrees loses no geometric information.

Kämpe, Sintorn, and Assarsson showed that reducing an SVO to a minimal DAG cuts node counts by one to three orders of magnitude for real scenes [2]. Their HAIRBALL benchmark — a dense tangle of curves with no obvious repetition — still saw a 28× reduction; scenes with regular geometry (CRYSPONZA) saw up to 576× reduction.

Traversal of the DAG is nearly identical to traversal of the SVO. The traversal loop does not know or care whether a child pointer is unique or shared — it just follows it [2]. The only difference is that you no longer get spatial context from the node's position in the tree: since a node may be reached via multiple paths, its position in the scene depends on how you arrived at it, not on which node it is. The traversal maintains the current world-space position in its own state, which the SVO traversal already does. See [sparse voxel DAGs](../storing/sparse-voxel-dags.md) for how the DAG is built and stored.

---

## GPU considerations

### the divergence problem

On a GPU, a warp of 32 threads executes the same instruction simultaneously. Ray traversal is inherently divergent: each ray descends into a different child, pops at a different level, and terminates at a different depth. Threads in the same warp quickly take different paths through the tree, leaving many SIMD lanes idle while others continue.

Mitigations that have been explored:

- **ray packet reordering** — group rays by current octree node and reorder into new packets of coherent rays periodically; expensive bookkeeping, but reduces dead lanes in complex scenes
- **beam optimization as coherence seed** — by giving blocks of 8×8 pixels the same starting depth, neighboring pixels stay in the same subtree for more steps before diverging
- **use the hierarchy for secondary rays** — shadow and ambient-occlusion rays (which need only a hit/miss answer, not the exact distance) can stop much earlier; Kämpe et al. shoot 64 shadow rays per pixel against the DAG and report 170 MRays/s and 240 MRays/s for hard and soft shadows respectively at 128K³ resolution on an NVIDIA GTX680 [2]

### memory bandwidth

The dominant cost after divergence is memory traffic. Each PUSH requires fetching the child descriptor of the new node. In the ESVO, child descriptors are 64 bits (a 15-bit child pointer, valid and leaf masks, and contour data) [1]. The DAG compresses this further since shared nodes are fetched once and reused across all paths that point to them.

Crassin et al.'s GigaVoxels system adds an LRU brick cache on the GPU: only the nodes visible from the current viewpoint need to be in GPU memory [3]. When a traversal step finds a node whose brick is missing, it flags that node for streaming from CPU memory and uses a coarser mipmap level instead. This is <em>ray-guided streaming</em> — the traversal itself drives what data gets loaded, so out-of-core datasets far larger than GPU memory become feasible. See [GPU voxel techniques](../optimization/gpu-voxel-techniques.md) for the caching and streaming patterns.

---

## tradeoffs: when to use this, when not to

### vs uniform-grid DDA

A [uniform-grid DDA traversal](./grid-ray-traversal.md) is simpler to implement and more GPU-friendly — all threads step by the same rule, divergence is low, and cache locality is predictable. It wins when:

- the scene fits in a manageable grid resolution (up to ~512³)
- the voxels are dense (few empty cells — empty-space pruning provides little benefit)
- the scene updates dynamically (rebuilding or modifying an octree is expensive; writing to a grid cell is O(1))

SVO ray casting wins when:

- the scene is huge and sparse (an outdoor world, an architectural scan)
- you need sub-pixel geometric detail — the tree's LOD delivers this without a separate system
- memory is the binding constraint — the SVO or DAG may fit in GPU memory where a dense grid would not

| | uniform grid DDA | SVO ray casting |
|---|---|---|
| empty space | steps one cell at a time | skips entire subtrees |
| memory | O(N³) | O(surface area) |
| dynamic edits | O(1) per voxel | tree rebuild or localized refit |
| LOD | needs a separate mipmap | free from the hierarchy |
| GPU coherence | high (uniform steps) | low (divergent paths) |
| implementation complexity | low | medium–high |

### the static geometry constraint

The strongest practical limitation of SVO ray casting is that it is best suited to static or slowly-changing geometry. Building the SVO from a mesh takes seconds to minutes depending on resolution and scene size. Modifying a single voxel requires updating all ancestor nodes along the path from that voxel to the root — not prohibitive for small edits, but not the casual O(1) write that a dense grid provides.

For scenes that need frequent destruction, carving, or procedural updates, the uniform grid or a hybrid structure (see [choosing a render path](./choosing-a-render-path.md)) is usually the better starting point. The SVO shines when the geometry is authored or scanned once and rendered many times — exactly the use case for architectural visualization, film VFX environments, and high-detail static world geometry.

For a full comparison of how this path fits alongside rasterization, ray marching, and splatting, see [ways to render voxels](./ways-to-render-voxels.md).

---

## references

[1] Laine, S. and Karras, T. (2010). "Efficient Sparse Voxel Octrees." *Proceedings of the ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (I3D 2010)*, pp. 55–63. DOI: 10.1145/1730804.1730814. [local PDF](../papers/laine-karras-2010-efficient-sparse-voxel-octrees.pdf) · [source](https://research.nvidia.com/publication/2010-02_efficient-sparse-voxel-octrees)

[2] Kämpe, V., Sintorn, E., and Assarsson, U. (2013). "High Resolution Sparse Voxel DAGs." *ACM Transactions on Graphics*, 32(4), Article 101. DOI: 10.1145/2461912.2462024. [local PDF](../papers/kampe-sintorn-assarsson-2013-high-resolution-sparse-voxel-dags.pdf) · [source](https://www.cse.chalmers.se/~uffe/HighResolutionSparseVoxelDAGs.pdf)

[3] Crassin, C., Neyret, F., Lefebvre, S., and Eisemann, E. (2009). "GigaVoxels: Ray-Guided Streaming for Efficient and Detailed Voxel Rendering." *Proceedings of the ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (I3D 2009)*, pp. 15–22. [local PDF](../papers/crassin-2009-gigavoxels-ray-guided-streaming.pdf) · [source](https://maverick.inria.fr/Publications/2009/CNLE09/)
