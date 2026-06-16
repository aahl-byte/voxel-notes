<link rel="stylesheet" href="./css/globals.css">

# the performance budget

The goal is a voxel system that holds a target frame rate and fits in a memory envelope — simultaneously, under real conditions, not just in a demo scene. Getting there requires spending optimization effort on the thing that is actually slow, not on the thing that is easiest to reach for. A voxel system that holds 60 fps on flat terrain but grinds on any view with depth is slow for a specific reason. A system that runs smoothly until a player starts editing is slow for a different reason. Optimizing the wrong thing doesn't just fail to help — it can make the other bottleneck worse by shifting more load onto it.

This page maps where time and space actually go in a voxel system, names those cost centers, and then routes each one to the technique that addresses it.

---

## the coarse model — three kinds of slow

Before measuring anything, it helps to know what the three fundamental constraints look like in practice.

A voxel system is <em>memory-bound</em> when the GPU runs out of VRAM and has to stall, spill to slower memory, or stream data over PCIe. The voxel problem is naturally prone to this: a dense grid scales as O(n³), so doubling linear resolution multiplies data by eight. A 512³ grid of 4-byte floats already occupies 512 MB before any shading data is attached. Sparse structures help, but even a sparse octree with 2.7 GB of scene data — like the Sibenik cathedral scene benchmarked by Laine and Karras — has to be managed carefully to stay within GPU memory bounds [1].

A system is <em>bandwidth-bound</em> when the GPU has data in memory but can't move it to its compute units fast enough. Memory bandwidth is distinct from memory capacity: you can have plenty of VRAM and still stall because the bus between the memory chips and the shader cores is saturated. Voxel ray casting and ray marching both hammer bandwidth because they read scattered locations across a large volume on every frame.

A system is <em>compute-bound</em> when the shader cores are the bottleneck — they have data and time to fetch it, but there are too many arithmetic operations to do. Dense ray marching through a full volume is the most compute-heavy path; hierarchical traversal that skips empty space trades some compute for more irregular memory access.

These three modes need different fixes. Compressing the voxel representation helps a memory-bound system but does nothing for a compute-bound one. Reducing shader complexity helps a compute-bound system but leaves a bandwidth-bound one unchanged. The first job of any optimization pass is to determine which mode you are actually in — and that means measuring, not guessing.

---

## measure first

The universal rule: **profile before optimizing.**

A frame profiler (RenderDoc, NSight, PIX) will show where time is actually going. The key questions to ask of the data:

- Is GPU utilization high while the CPU idles, or vice versa? High GPU utilization with a waiting CPU means the GPU is the bottleneck.
- Is GPU time dominated by memory fetches (cache misses, stalls) or by shader execution? GPU vendor tools expose both. A kernel with low arithmetic intensity and high memory stall counts is bandwidth-bound.
- Is CPU time dominated by draw call submission or by background work (meshing, generation, physics)? Too many draw calls makes the CPU the bottleneck even when the GPU has capacity to spare.

The Voxel Plugin documents this approach directly: `stat VoxelCounters` shows the number of triangles drawn and the number of draw calls, which together reveal whether a scene is GPU-limited by rasterization load or CPU-limited by submission overhead [4]. Only after measurement does it make sense to reach for a specific lever.

---

## the cost centers

### memory and bandwidth

The raw data volume is the foundational constraint in any voxel system. The O(n³) growth means that even moderate resolutions produce large grids. Moving that data between CPU and GPU — whether for uploads on edit, for streaming as the camera moves, or simply for the GPU to read during rendering — is where bandwidth is consumed.

Two distinct bandwidth budgets matter:

- **GPU memory bandwidth** — the rate at which the GPU can read from its own VRAM. This is the primary bottleneck for ray casting and ray marching. Both techniques read scattered voxel samples on every frame, and poor spatial locality in the data layout means frequent cache misses. GigaVoxels identified this directly: even when a volume technically fits in GPU memory, transferring 512 MB of data to the GPU each frame already prevents real-time performance [2]. Their solution — streaming only the visible, resolution-appropriate subset based on ray feedback — is a direct response to bandwidth pressure.
- **PCIe upload bandwidth** — the rate at which new data can move from CPU RAM to VRAM. This becomes the bottleneck during streaming or editing. PCIe 3.0 tops out around 16 GB/s; PCIe 4.0 around 32 GB/s. A voxel streaming system that pushes too many updated chunks per frame will saturate this bus and cause visible pop-in or frame stalls.

The lever here is [memory layout and data compression](./memory-layout-and-morton.md). Morton-order (Z-curve) storage improves 3D spatial locality for GPU cache, converting what would be scattered reads across a flat array into a more coherent access pattern. Compression (explored in [compression techniques](./compression-techniques.md)) reduces the raw byte count that has to move across both buses.

### draw calls and overdraw

When a voxel system uses the mesh-based render path — extract a surface mesh, rasterize it — it faces the same draw-call problem as any mesh renderer, but at higher volume. A chunked world with 16³-voxel chunks can produce thousands of chunk meshes, each requiring a separate draw call. Too many draw calls makes the CPU the bottleneck: the driver is busy submitting work while the GPU waits.

Overdraw is the complementary problem on the GPU side: rasterizing pixels that will be immediately discarded because something closer covers them. A voxel scene with many layers of geometry can rasterize the same screen pixel dozens of times, wasting fragment shader time.

The levers are [LOD and culling](./lod-and-culling.md): frustum culling removes chunks outside the camera frustum before they produce draw calls; occlusion culling skips chunks hidden behind opaque geometry; merging adjacent small chunks into larger meshes reduces call count at the cost of coarser per-chunk culling. Increasing the mesh block size from 16³ to 32³ can reduce draw calls by up to 4× in flat scenes — at the cost of slower per-edit remeshing and reduced culling granularity [4].

### traversal cost

For systems that use ray casting or ray marching — direct rendering without a surface mesh — the bottleneck is how many voxels each ray visits before hitting a surface or exiting the volume. A naive ray marching pass through a dense 512³ grid visits up to 512 voxels per ray per pixel. At 1080p that is roughly half a billion voxel reads per frame, most of which return "empty."

Hierarchical traversal is what makes direct voxel rendering viable at scale. By organizing voxels into a sparse octree or similar structure, a ray can skip entire empty subtrees in one step rather than testing them cell by cell. The DDA (digital differential analyzer) algorithm — which the Amanatides and Woo paper established as the foundation for regular grid traversal — steps the ray efficiently from voxel face to voxel face, but it cannot skip empty regions. Hierarchical methods like those used in ESVO can skip whole subtrees, dramatically reducing step count in sparse scenes [1].

The cost is pointer-chasing through the tree, which generates irregular memory access and hurts the GPU's cache — the bandwidth problem reappears in a different form. The lever is [GPU voxel techniques](./gpu-voxel-techniques.md): hierarchical ray marching, per-tile empty-space skipping, and beam optimization all reduce cells visited per ray.

### meshing and generation CPU cost

In systems that extract a mesh from the voxel field — marching cubes, dual contouring, surface nets — remeshing a chunk after an edit is a CPU-side cost. For a 16³ chunk, a fast SIMD marching cubes implementation achieves roughly 0.2–0.5 ms per chunk [3]. That sounds cheap; the problem is that a single edit can dirty dozens of adjacent chunks (especially near chunk borders), all of which need remeshing before the world looks correct.

Procedural generation compounds this: generating the voxel field for a new chunk from scratch (noise functions, signed-distance fields, rule-based layers) can cost more than meshing it. The Voxel Plugin's measurements show that on flat worlds, nearly 50% of generation time goes to querying the world generator — before any meshing work begins [4].

The levers are batching and threading: generation and meshing happen on background threads; only the mesh upload to the GPU touches the main thread. Range analysis — estimating which subregions of a chunk are guaranteed to be entirely inside or outside the surface — lets the mesher skip uniform blocks entirely. These concerns connect directly to [anatomy of a voxel engine](../engines/anatomy-of-a-voxel-engine.md), which covers how background threading and chunk scheduling are structured.

### upload cost (PCIe)

A separate cost center worth naming explicitly: even after a mesh or updated voxel block has been produced on the CPU, it has to transfer to VRAM over the PCIe bus before the GPU can use it. On some configurations the first mesh upload call during a frame can take up to 15 ms on the CPU — nearly an entire frame budget at 60 fps — as the driver stalls to process the transfer [5].

The fix is to limit uploads per frame (spread them across multiple frames), prefer asynchronous buffer uploads where the API supports them, and prioritize by visibility: chunks currently in the camera frustum upload first. The [storing domain](../storing/the-storage-problem.md) covers the data structures that minimize how much data needs to change per edit.

---

## bottleneck to lever — a routing map

| what you measure | bottleneck | fix lives here |
|---|---|---|
| GPU VRAM full, streaming stalls | memory capacity | [compression techniques](./compression-techniques.md), [storing](../storing/the-storage-problem.md) |
| GPU cache misses, scattered reads | memory bandwidth | [memory layout and Morton](./memory-layout-and-morton.md) |
| too many draw calls, CPU-limited | draw call overhead | [LOD and culling](./lod-and-culling.md) |
| fragment shader running excess pixels | overdraw | [LOD and culling](./lod-and-culling.md) |
| ray marching visits too many voxels | traversal cost | [GPU voxel techniques](./gpu-voxel-techniques.md) |
| meshing or generation too slow | CPU compute | threading, range analysis — [anatomy of a voxel engine](../engines/anatomy-of-a-voxel-engine.md) |
| frame stutters on chunk load | PCIe upload | amortize uploads, async transfer |
| lighting stale after edits | lighting recompute | [baking ambient occlusion and light](./baking-ambient-occlusion-and-light.md) |

---

## the tradeoff triangle

Every optimization in a voxel system navigates three competing pressures:

- **memory** — how much storage the representation requires (VRAM, system RAM, disk)
- **speed** — how fast the system renders, edits, or queries
- **quality** — resolution, geometric detail, lighting accuracy

Compressing voxel data reduces memory but costs decompression time during traversal (memory vs speed). Increasing chunk resolution improves quality but multiplies memory and remeshing cost (quality vs memory and speed). Pre-baking lighting produces high visual quality at low runtime cost but goes stale the moment geometry changes — speed and quality in the frame, at the cost of edit latency.

No technique moves all three vertices favorably at once. The voxel pipeline describes [the overall lifecycle](../foundations/the-voxel-pipeline.md) these tradeoffs play out across. Understanding which vertex your system is currently sacrificing — and why — is what the performance budget is ultimately for.

---

## references

[1] Laine, S. and Karras, T. (2010). "Efficient Sparse Voxel Octrees." *Proceedings of the 2010 ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (I3D '10)*, 97–105. DOI: 10.1145/1730804.1730814. [local PDF](../papers/laine-karras-2010-efficient-sparse-voxel-octrees.pdf) · [source](https://research.nvidia.com/publication/2010-02_efficient-sparse-voxel-octrees)

[2] Crassin, C., Neyret, F., Lefebvre, S., and Eisemann, E. (2009). "GigaVoxels: Ray-Guided Streaming for Efficient and Detailed Voxel Rendering." *Proceedings of the 2009 ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (I3D '09)*, 15–22. DOI: 10.1145/1507149.1507152. [local PDF](../papers/crassin-2009-gigavoxels-ray-guided-streaming.pdf) · [source](https://maverick.inria.fr/Publications/2009/CNLE09/)

[3] BonsaiRobo (2021). "Smooth Voxel Mapping: a Technical Deep Dive on Real-time Surface Nets and Texturing." *Medium / DreamCat Games*. [source](https://bonsairobo.medium.com/smooth-voxel-mapping-a-technical-deep-dive-on-real-time-surface-nets-and-texturing-ef06d0f8ca14)

[4] Voxel Plugin Documentation (2024). "Performance and Profiling." *docs.voxelplugin.com*. [source](https://docs.voxelplugin.com/1.2/technical-notes/performance-and-profiling)

[5] Zylann (2024). "Performance." *Voxel Tools for Godot — ReadTheDocs*. [source](https://voxel-tools.readthedocs.io/en/latest/performance/)
