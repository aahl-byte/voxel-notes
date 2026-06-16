<link rel="stylesheet" href="./css/globals.css">

# gpu voxel techniques

A game world holds tens of millions of voxels. Every frame, some of them need new geometry built, some need to be tested for visibility, and the ones that are visible need to be ray-marched or rasterized. On a CPU, even a fast one, those jobs are done by a handful of cores working sequentially through the list. On the GPU, the same jobs can run across thousands of execution units in parallel — one thread per voxel cell, one thread per chunk, one thread per screen tile — finishing in a fraction of the time.

That is the through-line for this page: moving the heavy voxel workloads off the CPU and onto the GPU, and understanding what it takes for those workloads to run fast once they are there.

Before diving in: [the performance budget](./the-performance-budget.md) explains why voxel work is expensive at scale. The memory layout choices that determine how efficiently the GPU can read voxel data are covered in [memory layout and Morton codes](./memory-layout-and-morton.md).

---

## the coarse picture

A voxel engine has three recurring jobs that eat the most time:

- **building geometry** — converting a chunk of voxel data into drawable triangles
- **traversal and tracing** — stepping a ray through a grid or tree to find what it hits
- **culling** — deciding which chunks and voxels are even worth processing this frame

All three are embarrassingly parallel: each chunk, each ray, each tile of the screen is independent of the others. That makes them a natural fit for the GPU, which was built around the idea of running thousands of independent threads at once.

The pattern that shows up repeatedly is: launch a compute dispatch with one thread (or one thread group) per unit of work, let the GPU process everything in parallel, and let the results sit in GPU-side buffers — never read them back to the CPU unless you absolutely must.

---

## the workloads that move to the GPU

### generating geometry — compute shader meshing

When a chunk changes — a block is placed, terrain is generated — its visible faces need to be turned into triangles. On the CPU this is a serial loop: examine each voxel cell, emit quads for exposed faces, optionally merge adjacent identical faces (greedy meshing). For a single chunk of 32³ cells that loop touches up to 32,768 cells, and in a large world hundreds of chunks may need rebuilding in one frame.

The GPU version launches a <em>compute shader</em> for meshing: one thread group per chunk, one thread per cell (or per face axis). Each thread examines its cell and its neighbors, decides whether a face is exposed, and writes the result into a shared output buffer — all chunks in parallel, all cells within a chunk in parallel [1]. A two-pass approach is common: a first dispatch counts how many vertices each chunk will emit (using atomic counters), and a second dispatch writes the actual vertex data into the correct offsets. The finished vertex buffers stay on the GPU and are drawn directly from there, never copied to the CPU [2].

GPU meshing on modern hardware is dramatically faster than CPU meshing: implementations on high-end GPUs report meshing rates well above 1000 frames per second for single chunks, versus 50–200 microseconds per chunk for optimized CPU greedy meshing [1]. The more important gain is parallelism across chunks: while the CPU rebuilds one chunk at a time (or at best one per CPU core), the GPU rebuilds all dirty chunks simultaneously in a single dispatch.

### tracing rays on the GPU — traversal and SVO ray casting

Ray traversal through voxel data is another naturally parallel job: each ray is independent. On the GPU every ray in a screen-sized batch can be traced simultaneously in a compute dispatch or a ray-gen shader.

For a flat uniform grid, the traversal algorithm itself is a straightforward DDA (digital differential analyzer) — the same grid stepping described in [grid ray traversal](../rendering/grid-ray-traversal.md). For sparse structures the ray must descend a tree. Laine and Karras [3] showed how to cast rays through a <em>sparse voxel octree</em> entirely on the GPU: the ray starts at the root, tests the current node's children using a packed child descriptor, descends into non-empty children, and steps past empty ones — all in a tight shader loop. Details of that algorithm are in [sparse voxel octree ray tracing](../rendering/sparse-voxel-octree-raytracing.md); the key GPU point is that it runs as a compute shader with one thread per ray, and at the tested resolutions achieved 60–122 million primary rays per second [3].

GigaVoxels [4] took a different approach: rather than a static tree, it streams voxel bricks on demand, guided by which rays are actually cast each frame. The GPU drives both the rendering (ray marching through an octree of bricks) and the request list (which bricks need to be loaded from disk or CPU memory) — the CPU only serves data the GPU asks for.

More recent work on wider branching trees (64-trees instead of 8-trees) achieves roughly 3× better memory efficiency than SVOs by packing child presence bitmasks into 64-bit integers, enabling faster coalesced reads during traversal [5].

A real challenge for GPU ray traversal is <em>warp divergence</em>: threads within a 32-thread warp that take different code paths (some rays hit geometry early, some step through empty space for many iterations) serialize rather than run in parallel. Wavefront path tracing — splitting the algorithm into separate kernels each doing uniform work — is one mitigation; another is choosing data structures whose traversal paths tend to be similar for nearby rays (which share a tile of the screen and therefore point in similar directions) [6].

### voxelizing meshes on the GPU — rasterizer voxelization

Sometimes the direction is reversed: you start with a triangle mesh and need to know which voxels it occupies. [Mesh voxelization](../generating/mesh-voxelization.md) covers this in full. The GPU path works by repurposing the rasterizer: project each triangle along the dominant axis (the one most perpendicular to the triangle face), let the fragment shader fire for each voxel cell the projected triangle covers, and write a "filled" flag into a 3D buffer via an atomic operation.

Standard rasterization only generates fragments at pixel centers, so thin triangles can slip between sample points. <em>Conservative rasterization</em> widens each triangle slightly in clip space (via the geometry shader, or via a hardware extension on newer GPUs) so that any cell even partially overlapped by the triangle generates a fragment. This guarantees no voxels are missed [7].

Schwarz and Seidel [7] showed this approach running at up to an order of magnitude faster than CPU voxelization for surface filling, and also described solid voxelization — filling the interior, not just the surface — using a tile-based parity accumulation on the GPU.

Crassin and Green [8] extended this to build a sparse voxel octree directly during the rasterizer pass: rather than filling a dense 3D buffer, each fragment atomically allocates and populates octree nodes, constructing the hierarchical structure in a single GPU-side pass.

---

## keeping the CPU out of the loop

### building draw commands on the GPU — indirect draw and dispatch

Once geometry is built on the GPU, the CPU still needs to issue draw calls — one per visible chunk, with the vertex count and buffer offset for each. At the scale of 400,000 chunks, that is 400,000 separate CPU calls per frame, which is far too many [9].

The solution is to store the draw arguments — vertex count, instance count, buffer offset — in a GPU buffer and let the GPU fill them in. The CPU then issues a single `ExecuteIndirect` (D3D12) or `vkCmdDrawIndirect` (Vulkan) call that reads its arguments from that buffer. This is called <em>indirect draw</em>.

The same mechanism applies to compute dispatches: <em>indirect dispatch</em> lets the GPU decide how many thread groups to launch for the next pass based on the output of the previous one — no CPU readback needed. Aokana [10] uses this throughout its pipeline: the tile selection pass fills a buffer of tile-chunk pairs, and an indirect dispatch compute shader immediately consumes that buffer to launch the ray-marching pass with exactly as many thread groups as there are tiles — all without the CPU ever seeing those counts.

### GPU-driven culling

With indirect draw in place, culling can also move to the GPU. A compute shader runs one thread per chunk, tests the chunk's bounding box against the current frustum (and optionally against a hierarchical depth buffer — Hi-Z), and either writes a draw command into the indirect buffer or skips it. Chunks that fail the test simply produce no draw command and cost no rasterization work [9][10].

Aokana's pipeline runs two selection passes: an initial frustum + Hi-Z cull, then a second pass that re-tests previously culled tiles with the current frame's Hi-Z texture to recover chunks that became visible mid-frame [10]. The CPU's involvement: zero. It sets up the passes at the start of the frame and submits.

### mesh shaders for voxel meshlets

The traditional rasterization pipeline has a fixed entry point: a vertex buffer assembled by the CPU (or an indirect command), then a vertex shader per vertex. <em>Mesh shaders</em> (DirectX 12 Ultimate / `VK_EXT_mesh_shader`) remove that fixed entry point entirely. A mesh shader is a compute-like shader that directly outputs vertices and triangles — up to about 256 vertices and 512 primitives per thread group [11].

For voxels, this fits naturally: a thread group takes a chunk (or a sub-chunk "meshlet"), generates its geometry on the fly in the shader, and feeds triangles directly to the rasterizer — no pre-built vertex buffer needed. The preceding <em>task shader</em> (called the amplification shader in D3D12) can cull entire meshlets — whole groups of voxel faces — before the mesh shader even runs, combining generation and culling in one GPU-side step [11][12].

The meshlet size for voxels is a tuning decision: a 16³ sub-chunk produces at most 16³ × 6 = 24,576 faces, far above hardware limits, so in practice a meshlet covers a smaller region or a fixed face budget. The task shader counts faces in the region first and spawns only the required mesh shader invocations.

---

## the rules that govern GPU speed

### coalesced memory access

A GPU processes 32 threads at a time in a group called a warp. When those 32 threads read memory, the hardware coalesces them into as few DRAM transactions as possible. If thread 0 reads address 0, thread 1 reads address 4, thread 2 reads address 8, and so on in a contiguous stride, all 32 reads merge into a single 128-byte transaction [13]. If the same 32 threads read 32 scattered, unrelated addresses, the hardware must issue up to 32 separate transactions, using 32× more bandwidth for the same data.

For voxel data, this means memory layout matters enormously. A 3D grid stored in naïve `[x][y][z]` row-major order is fine when threads march along the X axis — but as soon as they need Y or Z neighbors, the addresses scatter. <em>Morton (Z-order) interleaving</em> reorders voxel cells so that cells near each other in 3D space are also near each other in memory, improving locality in all three axis directions simultaneously. The mechanics are in [memory layout and Morton codes](./memory-layout-and-morton.md).

### occupancy and divergence

<em>Occupancy</em> is the fraction of a GPU's execution slots that are actually filled with active threads. High occupancy lets the GPU hide memory latency: while one warp waits for data to arrive from VRAM, the scheduler switches to another warp that is ready. Low occupancy means those idle cycles sit empty.

Occupancy is limited by shared memory and register use per thread: a shader that uses many registers can only fit fewer threads per streaming multiprocessor. For compute-shader meshing, keeping the per-thread register count low is a real design constraint.

<em>Divergence</em> is the enemy of occupancy for ray-tracing workloads: rays that terminate early leave threads idle within their warp while others keep stepping. Sorting rays by direction (so rays in the same warp point in similar directions and hit geometry at similar depths) reduces divergence. Grouping rays by screen tile achieves this cheaply — nearby pixels naturally have nearby directions [5][6].

### keeping data in VRAM

Every time voxel data crosses the PCIe bus — CPU uploads a new chunk, a readback copies results to CPU memory — a latency cost and a bandwidth cost appear. The GPU's VRAM is typically 10–100× faster to read than data coming over PCIe.

The key strategy is to keep voxel data resident on the GPU as long as possible. NanoVDB [14] is a GPU-native format for sparse VDB volumes: it serializes an OpenVDB tree into a flat, pointer-free buffer that can be copied to VRAM with a single `memcpy` and traversed entirely by GPU shaders without pointer chasing. Details are in [OpenVDB and NanoVDB](../storing/openvdb-and-nanovdb.md).

Brick pool architectures serve the same goal for large scenes: the world is divided into fixed-size bricks (e.g. 8³ or 16³ voxel sub-volumes), and a pool of bricks lives permanently in VRAM as a 3D texture atlas. A sparse indirection table maps world-space coordinates to brick addresses. When new terrain loads in, only the new bricks are uploaded — the rest stay resident and are sampled by shaders via texture hardware, which handles cache and trilinear filtering for free [4][15].

---

## tradeoffs — when the GPU approach costs more

### readback latency

If anything downstream needs to know what the GPU produced — the CPU wants a physics mesh, the game logic wants a voxel occupancy query — the results must be read back over PCIe. Readbacks are asynchronous and typically introduce a full frame (or more) of latency. Synchronous readbacks stall the GPU pipeline and can drop performance from thousands of frames per second to single-digit FPS [1][2].

The clean solution is to keep physics and logic queries GPU-side as well (collision detection via compute, voxel queries via GPU buffers). When CPU readback is genuinely required, use asynchronous readback with a per-frame budget and accept the latency.

### limited VRAM

VRAM is fast but finite — typically 8–24 GB on current hardware. A naïve dense grid at sufficient world resolution overflows it quickly (a 512³ grid of 4-byte values costs 512 MB; a 1024³ grid costs 4 GB). Sparse structures (SVOs, brick pools, NanoVDB, SVDAGs) exist precisely to keep resident data within budget. Streaming — evicting distant bricks, loading nearby ones — manages the working set at a PCIe cost that must stay within the frame budget.

### harder to debug

GPU shaders lack the debugger experience of CPU code. A misbehaving compute meshing shader produces garbled geometry with no stack trace. Tools like RenderDoc, Nsight, and GPU validation layers help, but the iteration loop is slower. This is a real cost when the algorithm is still being developed.

### when CPU meshing still wins

- **Low chunk count** — if the world has fewer than a few hundred dirty chunks per frame, the overhead of a compute dispatch and its synchronization can exceed the time saved.
- **Irregular meshing logic** — algorithms with complex conditional branching (e.g. per-voxel material-specific mesh generation) can diverge heavily on the GPU, making a single-threaded CPU loop competitive.
- **Immediate CPU access required** — physics engines that need triangle geometry synchronously are better served by a CPU mesher that produces data without a readback round-trip.
- **Simple prototyping** — a CPU greedy mesher in ~100 lines is far easier to get right first; GPU compute meshing can replace it once the algorithm is stable.

| situation | prefer |
|---|---|
| millions of voxels, large view distance | GPU compute meshing + indirect draw |
| small scene, few dirty chunks per frame | CPU meshing |
| ray-based rendering (no geometry) | GPU traversal compute shader |
| rasterized pipeline, dense near-field | GPU meshing → mesh shader meshlets |
| mesh needed synchronously by physics | CPU meshing with async upload |
| volume larger than VRAM | GPU streaming + brick pool |

---

## references

[1] McDonald, N. (2021). "High Performance Voxel Engine: Vertex Pooling." Personal blog. [source](https://nickmcd.me/2021/04/04/high-performance-voxel-engine/)

[2] Unity Community / artnas. "VoxelMeshGPU." GitHub repository. [source](https://github.com/artnas/unityvoxelmeshgpu)

[3] Laine, S. and Karras, T. (2010). "Efficient Sparse Voxel Octrees." *ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (I3D)*. DOI: 10.1145/1730804.1730814. [local PDF](../papers/laine-karras-2010-efficient-sparse-voxel-octrees.pdf) · [source](https://research.nvidia.com/publication/2010-02_efficient-sparse-voxel-octrees)

[4] Crassin, C., Neyret, F., Lefebvre, S., and Eisemann, E. (2009). "GigaVoxels: Ray-Guided Streaming for Efficient and Detailed Voxel Rendering." *ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (I3D)*. DOI: 10.1145/1507149.1507152. [local PDF](../papers/crassin-2009-gigavoxels-ray-guided-streaming.pdf) · [source](https://hal.science/inria-00345899)

[5] dubiousconst282. (2024). "A Guide to Fast Voxel Ray Tracing Using Sparse 64-Trees." Personal blog. [source](https://dubiousconst282.github.io/2024/10/03/voxel-ray-tracing/)

[6] enki Software. (2023). "Implementing a GPU Voxel Octree Path Tracer." Devlog. [source](https://www.enkisoftware.com/devlogpost-20230823-1-Implementing-a-GPU-Voxel-Octree-Path-Tracer)

[7] Schwarz, M. and Seidel, H.-P. (2010). "Fast Parallel Surface and Solid Voxelization on GPUs." *ACM Transactions on Graphics (SIGGRAPH Asia)*, 29(6). DOI: 10.1145/1882261.1866201. [local PDF](../papers/schwarz-seidel-2010-fast-parallel-voxelization-gpus.pdf) · [source](https://michael-schwarz.com/research/publ/2010/vox/)

[8] Crassin, C. and Green, S. (2012). "Octree-Based Sparse Voxelization Using the GPU Hardware Rasterizer." In *OpenGL Insights*, CRC Press. [local PDF](../papers/crassin-green-2012-octree-sparse-voxelization-gpu.pdf)

[9] vkguide.dev / Project Ascendant. "High-Performance Voxel and Mesh Rendering." Vulkan Guide. [source](https://vkguide.dev/docs/ascendant/ascendant_geometry/)

[10] Fang, Y., Wang, Q., and Wang, W. (2025). "Aokana: A GPU-Driven Voxel Rendering Framework for Open World Games." *Proceedings of the ACM on Computer Graphics and Interactive Techniques (I3D)*. DOI: 10.1145/3728299. [local PDF](../papers/fang-2025-aokana-gpu-driven-voxel-rendering.pdf) · [source](https://arxiv.org/abs/2505.02017)

[11] Microsoft DirectX Team. (2019). "Coming to DirectX 12 — Mesh Shaders and Amplification Shaders." DirectX Developer Blog. [source](https://devblogs.microsoft.com/directx/coming-to-directx-12-mesh-shaders-and-amplification-shaders-reinventing-the-geometry-pipeline/)

[12] AMD GPUOpen. "From Vertex Shader to Mesh Shader." [source](https://gpuopen.com/learn/mesh_shaders/mesh_shaders-from_vertex_shader_to_mesh_shader/)

[13] Bajaj, H. (2024). "Memory Coalescing in GPU." Medium / The Arch Bytes. [source](https://medium.com/@himanshu0525125/memory-coalescing-in-gpu-23f222b26ca2)

[14] Museth, K. (2021). "NanoVDB: A GPU-Friendly and Portable VDB Data Structure for Real-Time Rendering and Simulation." *ACM SIGGRAPH Talks*. [local PDF](../papers/museth-2021-nanovdb-gpu-friendly-portable-vdb.pdf) · [source](https://www.researchgate.net/publication/353747008_NanoVDB_A_GPU-Friendly_and_Portable_VDB_Data_Structure_For_Real-Time_Rendering_And_Simulation)

[15] GPU Volume Rendering / Brick Caching. (2025). "GPU Volume Rendering with Hierarchical Compression Using VDB." arXiv:2504.04564. [source](https://arxiv.org/abs/2504.04564)
