<link rel="stylesheet" href="./css/globals.css">

# openvdb and nanovdb

Every cloud, plume of smoke, and wall of fire you have seen in a major animated or visual effects film in the last decade was almost certainly stored in the same format. A fluid sim grows from a small seed to fill an enormous space; the geometry around it changes every frame; at peak it might cover hundreds of millions of voxels. The production pipeline needs to store that volume, stream it to a renderer, and let the simulation add and delete regions of space as the effect evolves — all without pre-allocating memory for a bounding box that could be kilometres wide.

The data structure behind this is a shallow, wide tree — a hash map at the root fanning out over large intermediate nodes, each of which fans out over small dense brick nodes that each hold a tight local grid of voxels. This structure earned a Scientific and Engineering Award from the Academy of Motion Picture Arts and Sciences in 2024. It is called <em>VDB</em>, and the open-source library built around it is **OpenVDB** [1].

For GPU rendering and simulation, there is a companion form: the entire VDB tree is baked into a flat, contiguous byte buffer with no pointers, which can be uploaded directly to a GPU and traversed in CUDA or HLSL. That form is <em>NanoVDB</em> [2].

---

## why this problem is hard

[The storage problem](./the-storage-problem.md) establishes the core tension: a dense grid that is large enough to hold a fully-grown fluid sim would need hundreds of gigabytes, but the actual occupied region is a thin shell — the rest is empty air. A naïve dense grid wastes almost all of it.

Earlier solutions exist. [Hash grids and bricks](./hash-grids-and-bricks.md) carve space into fixed-size brick chunks and only allocate occupied chunks; they are fast and cache-friendly but have no hierarchy — you cannot represent a uniform region more cheaply than an active one. [Octrees and SVOs](./octrees-and-svo.md) give you hierarchy and let you collapse large uniform subtrees to a single value, but they are deep (O(log N) levels for resolution N), and every tree traversal chases pointers down many levels, punishing the cache.

VDB is a synthesis: a shallow, wide tree that gets the sparsity of a hash structure and the uniform-region compression of a hierarchy, while keeping the traversal depth low enough to stay cache-friendly.

---

## the structure — a shallow, wide tree

Picture the tree from top to bottom.

**Root node — an unbounded hash map.** At the very top there is no fixed grid. Instead, the root holds a hash map keyed by the 3D index of each large region of space that exists. Adding a new region to the grid means inserting an entry into the hash map. This is why VDB is called *effectively unbounded*: the root does not pre-allocate a box; it grows on demand. Fast hash lookup means going from a world-space coordinate to the right subtree is a single map probe, not a traversal.

**Internal nodes — large, dense bitmask arrays.** Below the root are one or two levels of internal nodes. Each internal node covers a large cube of index space — in the standard configuration, the upper internal node covers 32 × 32 × 32 child slots; the lower covers 16 × 16 × 16 child slots. Crucially, each slot in an internal node can hold *either* a pointer to a child node *or* a constant tile value, tracked by a bitmask. When an entire 16³ or 32³ sub-region is uniform (all air, all solid, all the same density), it is collapsed to one tile value. No child node is needed, and no memory is allocated below it. This is how VDB achieves efficient storage for large uniform regions.

**Leaf nodes — small dense voxel bricks.** At the bottom, each leaf node holds an 8 × 8 × 8 (= 512 voxels) dense array of values plus a bitmask tracking which voxels are active. The leaf is where actual voxel data lives. Bitmasks are cheap to scan and allow SIMD-friendly operations across the 512-voxel block.

This four-level tree (root → internal 32³ → internal 16³ → leaf 8³) is what the paper calls the <em>5-4-3 configuration</em> — the numbers are the base-two logarithms of each level's branching factor, read from the top internal node down to the leaf (2⁵ = 32, 2⁴ = 16, 2³ = 8) [1].

```
root (hash map, unbounded)
 ├── tile: uniform region (whole 32³ sub-tree collapsed)
 ├── InternalNode 32³
 │    ├── tile: uniform sub-region
 │    └── InternalNode 16³
 │         ├── tile: uniform 16³ block
 │         └── LeafNode 8³  ← 512 voxels, dense array
 │              └── [voxel values + active bitmask]
 └── ...
```

The tree is only **four levels deep** regardless of the total resolution. An octree holding the same resolution with 2 children per axis at each split would need up to 20+ levels of traversal. Fewer levels means fewer pointer dereferences to reach a voxel, and fewer cache misses.

---

## tile values — sparsity and level-of-detail in one

A <em>tile value</em> is a constant value stored at an internal or root node slot to represent every voxel in that entire sub-tree uniformly. The FAQ definition is precise: "regions of index space with constant value and active state" [3].

This does two things at once:

- **Sparsity.** Empty air is a tile value at the highest applicable internal node. No leaf nodes are allocated for it; no memory is wasted.
- **Built-in level of detail.** A renderer or simulation can interrogate the tree at any level and receive a valid (coarser) answer. A cloud renderer at a great distance can sample internal node tile values instead of descending to leaves — effectively a built-in mip-map at the tree level.

---

## fast access — the value accessor

Going from a 3D world coordinate to its voxel value requires traversing root → internal → internal → leaf. Done naïvely on every lookup that is three pointer dereferences and a hash probe. For a fluid sim iterating over every active voxel in sequence this overhead accumulates.

VDB solves this with a <em>value accessor</em>: a per-thread accelerator object that caches the nodes visited on the last lookup. On a subsequent lookup that falls in the same leaf (or the same internal node), traversal starts from the cached node instead of the root. The documentation describes this as "inverted traversal" — the accessor checks its cache bottom-up first, and only climbs to the root if necessary [3]. For spatially coherent access patterns (neighbors in a sim stencil, sequential voxel iteration) the accessor consistently hits its cache and amortizes the traversal cost. The documentation reports typical speedups of about three times over uncached traversal.

This bottom-up cache strategy is the same idea used in B+ trees to speed range scans, and it is why the paper describes VDB as sharing characteristics with B+ trees [1].

---

## dynamic topology — why VFX simulations need it

A mesh does not change its polygon count mid-simulation. A fluid sim does: new smoke billows out from an explosion and fills previously empty space; old smoke dissipates and those regions go back to inactive. The grid topology changes every frame.

VDB supports this directly. The root's hash map can accept new entries; new internal nodes and leaf nodes can be allocated mid-sim; nodes can be pruned when a region returns to background value. The library provides merge, dilate, erode, and resample operations that all respect this dynamic topology. This is what the paper means by its subtitle "dynamic topology" [1] — the tree is not a static snapshot; it is a live structure that grows and shrinks with the simulation.

This was the key capability that hash grids and bricks already offered, but VDB adds it to a full hierarchy with tile-value compression and fast cached access.

---

## why VDB won in VFX

| property | dense grid | hash/brick grid | deep octree | VDB |
|---|---|---|---|---|
| effectively unbounded | no | yes | no | yes |
| tile-value compression | no | no | yes | yes |
| dynamic topology | no | yes | hard | yes |
| traversal depth | 1 | 1 | O(log N) | 4 levels |
| cache friendliness | excellent | good | poor | good |
| random access cost | O(1) | O(1) avg | O(log N) | O(1) avg |
| sequential access | trivial | good | complex | excellent (accessor) |

The combination of effectively unbounded grids, dynamic topology, and O(1) average access through the value accessor fit exactly the workflow of a fluid simulation feeding a path tracer. OpenVDB was first open-sourced by DreamWorks Animation in August 2012, became the first project hosted by the Academy Software Foundation (ASWF) in 2018, and received a Scientific and Engineering Award from the Academy in 2024 — with Ken Museth, Peter Cucka, and Mihai Aldén recognised for its creation, and Jeff Lait, Dan Bailey, and Nick Avramoussis for its continued evolution [4]. The citation reads: "OpenVDB's core voxel data structures, programming interface, file format and rich tools for data manipulation continue to be the standard for efficiently representing complex volumetric effects, such as water, fire and smoke." Today it is natively supported by Houdini, Arnold, RenderMan, Blender, Cinema 4D, Maya, and Unreal Engine, among others.

For VFX production specifics — which studios use it, how it fits into the rendering pipeline, and film-by-film examples — see [VDB in VFX](../applications/vdb-in-vfx.md).

---

## nanovdb — the gpu form

OpenVDB is a rich C++ library. Its tree uses heap-allocated nodes connected by pointers. Uploading that to a GPU means serialising a fragmented, pointer-laden structure, resolving pointers to GPU addresses, and linking against a library that assumes STL, exceptions, and dynamic allocation — none of which exist in GPU shader code.

NanoVDB solves this by converting an OpenVDB grid into a <em>single, flat, contiguous byte buffer</em> with no pointers. Every node in the buffer knows the byte offset to its children within the same buffer. The root sits at a known location at the front; internal nodes follow; leaf nodes at the end. To traverse the tree on a GPU thread, you read from the buffer and add offsets — no pointer chasing, no dynamic allocation, no library. The whole buffer can be copied with a single `cudaMemcpy` [2].

The trade-off is that the topology is frozen at conversion time. You can update voxel values in a NanoVDB buffer (they live in the leaf arrays), but you cannot add or remove nodes. NanoVDB is a <em>static snapshot</em> of whatever VDB grid you baked it from. This is fine for rendering and collision detection; it is not suitable for the simulation step itself, which needs to grow and shrink the grid.

The NanoVDB paper was presented at SIGGRAPH 2021 Talks (DOI: 10.1145/3450623.3464653). NVIDIA open-sourced the project and it is now part of the OpenVDB repository. Supported targets include CUDA, OpenCL, OpenGL, DirectX 12, OptiX, Vulkan, HLSL, and GLSL [2].

### nanovdb in practice

A typical pipeline looks like:

1. **Simulate** on CPU (or GPU with OpenVDB's growing toolkit) — topology changes freely.
2. **Convert** the frame's OpenVDB grid to NanoVDB using the provided tools.
3. **Upload** the NanoVDB buffer to the GPU.
4. **Render / query** in a CUDA or HLSL kernel — fast random access, cache-coherent traversal, no dependencies.

This means the same VDB hierarchy is used at both steps; NanoVDB is not a different data model, just a different memory layout of the same tree.

### vdb vs nanovdb at a glance

| | OpenVDB | NanoVDB |
|---|---|---|
| topology | dynamic — nodes added/removed freely | static after conversion |
| memory layout | heap-allocated, pointer-based, fragmented | single contiguous buffer, pointer-free |
| gpu compatible | no (C++ STL, dynamic alloc) | yes (C99/C++11, no dependencies) |
| modifiable values | yes | yes |
| modifiable topology | yes | no |
| primary use | simulation, geometry processing | rendering, collision on GPU |
| language targets | C++ only | CUDA, GLSL, HLSL, OpenCL, C++, C |

---

## where vdb sits among the options

VDB is not always the right choice. [Choosing a voxel store](./choosing-a-voxel-store.md) covers the full decision, but the short version:

- **Use VDB when** the volume is large, sparse, and changes topology over time — fluid sims, pyro, level sets, medical volumes with complex interiors.
- **Use a dense grid or chunk store** when the data is nearly full (e.g., a Minecraft-style world that is mostly occupied) — VDB's tree overhead is unnecessary and dense access is faster. See [dense grids and chunks](./dense-grids-and-chunks.md).
- **Use a hash/brick grid** when you need GPU-native dynamic allocation without a full library — the hash approach is simpler to implement and works well for real-time mapping. See [hash grids and bricks](./hash-grids-and-bricks.md).
- **Use NanoVDB** when you have a VDB grid that needs to go to the GPU for rendering or collision — it is the right bridge, not a replacement for the CPU-side simulation store.
- **Use an SVO or SVDAG** when the data is static, deeply compressible, and primarily read for rendering — SVDAGs compress repetitive geometry far below what VDB's tile values can do. Those tradeoffs are in [octrees and sparse voxel octrees](./octrees-and-svo.md) and [sparse voxel DAGs](./sparse-voxel-dags.md).

For GPU rendering acceleration on top of a VDB store, see [GPU voxel techniques](../optimization/gpu-voxel-techniques.md). For acquiring volumetric data that goes into VDB grids in the first place, see [scanned and volume data](../generating/scanned-and-volume-data.md).

---

## the specifics

### node dimensions in the 5-4-3 config

| level | name | dimension | child slots | voxels subsumed |
|---|---|---|---|---|
| root | RootNode | unbounded | hash map | all |
| 2 | InternalNode (upper) | 32³ | 32,768 | 32,768 × 16³ × 8³ |
| 1 | InternalNode (lower) | 16³ | 4,096 | 4,096 × 8³ |
| 0 | LeafNode | 8³ | 512 voxels | 512 |

Each slot at levels 1 and 2 holds either a child pointer or a tile value (tracked by a per-node bitmask). A single upper internal node tile covers 4096 × 4096 × 4096 voxels as one constant value.

### what a voxel can store

VDB is templated: a grid's voxel type is set at compile or instantiation time. Common payload types:

- **float** — density, level-set signed distance, temperature
- **Vec3f** — velocity, colour, surface normal
- **bool** — occupancy mask
- **int32** — material or object IDs

Each grid in an `.vdb` file is typed independently. A single file commonly packages a density grid, a velocity grid, and a flame grid for the same sim frame.

### the .vdb file format

OpenVDB has a versioned binary file format (`.vdb`) that is the interchange standard for volumetric VFX assets. NanoVDB can read and write `.nvdb` files (a headerless contiguous-buffer format) but also supports conversion to/from `.vdb`. Both formats travel with the notes: [museth 2013 VDB paper](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf) and [museth 2021 NanoVDB paper](../papers/museth-2021-nanovdb-gpu-friendly-portable-vdb.pdf).

---

## references

[1] Museth, K. (2013). "VDB: High-Resolution Sparse Volumes with Dynamic Topology." *ACM Transactions on Graphics*, 32(3), Article 27. DOI: 10.1145/2487228.2487235. [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf) · [source](https://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf)

[2] Museth, K. (2021). "NanoVDB: A GPU-Friendly and Portable VDB Data Structure For Real-Time Rendering And Simulation." *ACM SIGGRAPH 2021 Talks*. DOI: 10.1145/3450623.3464653. [local PDF](../papers/museth-2021-nanovdb-gpu-friendly-portable-vdb.pdf) · [source](https://history.siggraph.org/wp-content/uploads/2022/06/2021-Talks-Museth_NanoVDB.pdf)

[3] OpenVDB Documentation. "Frequently Asked Questions." Academy Software Foundation. [source](https://www.openvdb.org/documentation/doxygen/faq.html)

[4] "Sci-Tech winners include OpenVDB, Marvelous Designer, USD, Alembic." *befores & afters*, January 2024. [source](https://beforesandafters.com/2024/01/12/sci-tech-winners-include-openvdb-marvelous-designer-usd-alembic-and-the-blind-driver-roof-pod/)
