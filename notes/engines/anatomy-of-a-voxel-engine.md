<link rel="stylesheet" href="./css/globals.css">

# anatomy of a voxel engine

A player jumps into a game world, flies at speed over mountains that stretch to every horizon, digs a hole with a pickaxe, watches chunks of rock tumble away, and keeps moving — all at 60 frames per second, with no loading screen, no stutter, no seam. The world feels infinite. Nothing about it was in memory a minute ago, and the chunks behind the player are already gone.

That experience is the output of five subsystems cooperating across multiple threads every frame. Each subsystem owns a slice of the problem. They hand data to each other in a defined order. When one falls behind, the player feels it.

This page maps those subsystems and the flow of data between them. The individual pages in this domain go deep on each one — this is the overview you return to when you lose the thread.

For the four-stage pipeline these subsystems are built on, see [the voxel pipeline](../foundations/the-voxel-pipeline.md).

---

## the coarse model — five subsystems, one cycle

Before going into specifics: here is the mental model you should carry.

An interactive voxel engine is a cycle, not a pipeline. Data enters at the world boundary and flows inward toward the screen — but edits reverse the flow, feeding back into the store, re-triggering the forward pass.

The five subsystems, described plainly:

1. **Something holds all the voxel data that is currently in memory.** It knows which chunks exist, what values are in them, and where they live on disk or in a generator. This is the *store*.

2. **Something fills new chunks with content.** When the store needs a chunk that doesn't exist yet, it asks a function to produce voxel values for it — from noise, rules, authored data, or some mix. This is the *generator*.

3. **Something turns chunks of voxel data into geometry the GPU can draw.** It reads the voxel values, runs a meshing algorithm, and produces a list of triangles or quads. This is the *mesher*.

4. **Something draws those triangles on screen, applies lighting, and produces the final image.** This is the *renderer*.

5. **Something decides what the other four should be working on right now.** It watches the camera, calculates which chunks should exist in memory, which ones need to be generated, which generated chunks need meshes, and which distant chunks can be safely evicted to free space. This is the *streamer* (or *scheduler*).

These five things cooperate in a cycle. The cycle is what the player experiences.

---

## the five subsystems, described and named

### the voxel store — what it owns, who it talks to

The store is the authoritative in-memory record of the world. It holds all chunks that are currently loaded — each chunk is a fixed-size 3D grid of voxel values (occupancy, material, density, or whatever the engine stores per voxel). Chunks not currently in memory are either on disk or reconstructable by the generator.

The store is what every other subsystem reads from or writes to:

- The generator writes new chunk data into it.
- The mesher reads chunk data out of it.
- The renderer indirectly depends on it (via the meshes the mesher produced).
- The streamer tells it when to allocate a new chunk slot and when to drop one.

The store also handles the *dirty* state: any time a voxel value changes — because of an edit, a simulation step, or a generation update — the store marks the containing chunk (and its neighbors, if the mesher needs to see across chunk boundaries) as needing a new mesh. That dirty flag is what connects edits back to the mesher.

The data structures that back the store — flat grids, hash maps of chunks, sparse octrees, VDB — are covered in [dense grids and chunks](../storing/dense-grids-and-chunks.md).

### the generator — what it owns, who it talks to

The generator owns the procedure that turns a chunk coordinate into voxel values. It does not hold any state itself — it is a pure function of position (or close enough to one). When the streamer decides a new chunk needs to exist, it hands the chunk's world position to the generator, which fills the chunk's voxel grid and returns it to the store.

Generators are typically the most compute-heavy part of the pipeline per chunk — evaluating noise functions, running cellular automata, applying layered rules. They are always called off the main thread, often with many chunks being generated in parallel on a thread pool.

A generator can be:
- **procedural** — a mathematical function (noise, SDF, layered rules) that produces values from coordinates alone; infinitely scalable and reproducible.
- **authored/voxelized** — pre-baked data, or a mesh converted to voxel form, read from disk.
- **simulated** — values written by a running simulation, which means the generator and the simulation system overlap.

### the mesher — what it owns, who it talks to

The mesher reads voxel data from the store and produces renderable geometry. It is triggered whenever a chunk is marked dirty — a new chunk arrived, a voxel was edited, or a neighboring chunk changed in a way that affects this chunk's border faces.

The mesher owns only a transient working buffer: it reads in, runs the algorithm, writes triangles out. The output is a mesh ready to be uploaded to the GPU. It has no persistent state of its own.

The mesher typically sees not just the dirty chunk but a thin halo of neighboring voxels — one voxel deep on each face — so it can decide which boundary faces are visible and compute per-vertex ambient occlusion correctly. This is why marking a chunk dirty also marks its immediate neighbors as needing attention.

Meshing is CPU-heavy (especially greedy meshing on large chunks) and is always run off the main thread. The resulting vertex data is then uploaded to the GPU — and that upload step, where the data crosses from CPU memory to GPU memory, typically must occur on the main thread or on a thread that has the GPU context. This GPU upload is often a bottleneck in high-throughput pipelines.

### the renderer — what it owns, who it talks to

The renderer takes uploaded GPU meshes and draws them. It does not know or care about voxels — it sees a collection of meshes, each associated with a chunk position, and draws them using whatever render path the engine has chosen: rasterization, ray marching, or a hybrid.

The renderer owns:
- The GPU buffers (vertex data, draw commands).
- The render state (camera matrices, lighting, shadow maps).
- The culling logic — frustum culling drops chunks outside the camera frustum; occlusion culling skips chunks hidden behind others.

In engines that use indirect draw calls, the renderer hands a large buffer of draw commands to the GPU and lets a compute shader cull them — which keeps CPU-GPU synchronization minimal and scales to hundreds of thousands of chunks efficiently.

The renderer is the one subsystem that always runs on the main thread (or the GPU thread), since it must interact with the graphics API.

### the streamer / scheduler — what it owns, who it talks to

The streamer is the engine's executive. It runs on (or tightly coordinates with) the main thread and directs all the other subsystems. Every frame it answers a small set of questions:

- Which chunks should exist in memory given where the camera is?
- Of the chunks that should exist but don't, which ones need to be generated?
- Of the generated chunks, which ones have dirty meshes and need to be re-meshed?
- Of the chunks that are far enough away, which ones should be evicted to free memory?
- How many of each task can we dispatch this frame without causing a stall?

The streamer maintains a priority queue of work. Chunks close to the camera that are not yet loaded are the highest priority; distant chunks waiting for their second LOD refinement are low priority. When the camera moves quickly, the streamer has to reprioritize in real time, canceling in-flight work for chunks that are no longer relevant.

Chunk streaming and the scheduler's full decision space are covered in [chunk management and streaming](./chunk-management-and-streaming.md).

---

## the per-frame dataflow

### the steady state — camera moving through a loaded world

Each frame:

```
camera position
    │
    ▼
streamer calculates which chunks should be loaded
    │
    ├─► new chunks needed → queue to generator (background thread)
    │       generator fills voxel grid → chunk written to store
    │       chunk marked dirty
    │
    ├─► dirty chunks → queue to mesher (background thread)
    │       mesher reads voxels + neighbor halo → produces vertex buffer
    │       vertex buffer uploaded to GPU (main thread)
    │
    └─► distant chunks → evict from store (free memory)

renderer draws all chunks with uploaded meshes
    │
    └─► frustum + occlusion culling → draw calls → pixels on screen
```

In practice, all of these are in flight simultaneously — the generator is filling next-frame chunks while the mesher re-meshes the just-arrived ones from the frame before. The pipeline is not a waterfall; it is a continuously running set of parallel queues.

### the edit path — a player digs a hole

An edit (placing or removing voxels) is a short-circuit that re-enters the pipeline at the store:

```
player input → edit applied to store
    │
    ├─► affected chunk marked dirty
    │
    └─► six neighboring chunks may also be marked dirty
         (if the edit is on or near a chunk boundary)

mesher picks up dirty chunks → re-meshes → uploads to GPU
renderer draws updated mesh next frame
```

The key property that makes this fast: the mesher only re-meshes the affected chunk (and its immediate neighbors). It does not re-mesh the entire world. A single block dig should cause at most seven chunk re-meshes — one for the chunk itself and one for each of its six face neighbors. For small chunk sizes (16³ voxels), each re-mesh is measured in microseconds to low milliseconds. That budget is easily met within a single frame.

Runtime editing, its edge cases, and CSG operations are covered in [runtime editing and CSG](./runtime-editing-and-csg.md).

---

## why almost all of this runs off the main thread

The main thread has one job: produce a frame. On a 60fps target it has 16.67ms to do that. On a 30fps target, 33ms. Everything else competes for that budget.

Generation and meshing are the two expensive operations. On a dense chunk (full of varied voxels), meshing can take 100µs to several milliseconds. Generation with complex noise functions can take longer. If either ran on the main thread, flying over new terrain would cause visible frame drops — the main thread would stall waiting for a chunk to be generated and meshed before it could draw.

The solution is to push both operations to a thread pool and use the main thread only for:
- Updating the streamer (deciding what to queue).
- Uploading completed meshes to the GPU (a graphics-API requirement in most implementations).
- Running the renderer.

The constraint is that GPU uploads typically require access to the graphics context, which is bound to the main thread. This means the pipeline has a seam: the mesher finishes its work on a background thread, hands a buffer to the main thread, and the main thread uploads it — usually with a per-frame time budget so it doesn't blow the frame time even when many chunks arrive at once.

This threading architecture and its synchronization patterns are covered in [threading and the meshing pipeline](./threading-and-meshing-pipeline.md).

---

## the recurring tension — latency, memory, and detail

Every design decision in a voxel engine is a negotiation between three competing pressures. They cannot all be maximized at once.

### latency — no hitches

Latency is the time between "the player moved" and "the screen reflects it." Two kinds of latency matter:

- **Edit latency** — how long between a block dig and the updated mesh appearing on screen. Should be one or two frames (well under 100ms) or the world feels unresponsive.
- **Streaming latency** — how long before a newly visible region has its geometry. A few seconds of pop-in is tolerable; perpetual gray holes are not.

Reducing latency means doing work faster — smaller chunks (cheaper to re-mesh), more worker threads, smarter scheduling.

### memory — the world can't all fit

Voxel data scales with volume: double the render distance in each axis and you need eight times the memory. A 512³ region of 1-byte voxels is 128MB — just for the raw data, before meshes. Meshes are typically larger than the raw data.

The streamer evicts distant chunks to control memory use. The mesher can help by emitting only the minimal geometry (greedy meshing, omitting interior faces). Sparse storage structures (hash maps, VDB) skip empty space. But there is always a ceiling — the engine must decide how large a region around the camera to keep live, and stick to it.

### detail — the world should look good up close

High detail means small voxels and dense meshes. Small voxels mean more chunks per unit of space, more meshing work, more memory. This is why level-of-detail (LOD) systems exist: use coarse voxels for distant terrain, fine voxels close to the camera. The boundary between LOD levels must be stitched without visible cracks — a non-trivial meshing problem. LOD in engines is covered in [LOD in engines](./lod-in-engines.md).

| if you push here | you pay there |
|---|---|
| smaller chunks (lower edit latency) | more draw calls; more meshing overhead |
| larger render distance | more memory; more generation/meshing work |
| finer voxel resolution | more memory; longer meshing times |
| more aggressive LOD | lower memory; cracking artifacts at boundaries |
| bigger thread pool | higher CPU usage; contention with game logic |

There is no free lunch. Every shipped voxel game is a specific negotiation of this triangle, tuned to the target hardware and the gameplay goals. [The performance budget](../optimization/the-performance-budget.md) maps these tradeoffs in detail.

---

## the domain map — what lives where

This domain covers the engine layer: the subsystems and how they are wired together as a running system. The foundation domains cover what each subsystem works with.

| page | what it covers |
|---|---|
| [chunk management and streaming](./chunk-management-and-streaming.md) | how the streamer decides what to load, generate, mesh, evict; priority queues; camera-relative loading |
| [threading and the meshing pipeline](./threading-and-meshing-pipeline.md) | the thread pool architecture; GPU upload; synchronization; frame-budget limiting |
| [runtime editing and CSG](./runtime-editing-and-csg.md) | edit → dirty → re-mesh cycle; constructive solid geometry at runtime; undo/redo |
| [LOD in engines](./lod-in-engines.md) | level-of-detail schemes; transvoxel stitching; octree LOD; visual pop |
| [case studies](./case-studies.md) | how Minecraft, Teardown, and others made specific tradeoffs |

Behind this domain, the foundation concepts are:

| page | what it covers |
|---|---|
| [the voxel pipeline](../foundations/the-voxel-pipeline.md) | the four-stage lifecycle these engines are built on |
| [dense grids and chunks](../storing/dense-grids-and-chunks.md) | the data structures the store uses |
| [the performance budget](../optimization/the-performance-budget.md) | where the CPU/GPU time goes and how to control it |

---

## references

[1] Voxel Tools for Godot — Performance documentation. Zylann. https://voxel-tools.readthedocs.io/en/latest/performance/ (retrieved June 2026). Describes threading architecture, GPU upload bottleneck, SpatialLock3D synchronization, frame-budget limiting, and the latency impact of chunk size.

[2] McDougle, N. (2021). "High Performance Voxel Engine: Vertex Pooling." nickmcd.me. https://nickmcd.me/2021/04/04/high-performance-voxel-engine/ (retrieved June 2026). Details the GPU memory pool, DAIC indirect drawing, per-frame upload pipeline, and the meshing-vs-loading bottleneck split.

[3] Neicho, J. (blog). "Exile: Voxel Rendering Pipeline." thenumb.at. https://thenumb.at/Voxel-Meshing-in-Exile/ (retrieved June 2026). Illustrates the store → mesher → GPU dataflow and the tradeoff between meshing throughput and VRAM.

[4] Project Ascendant Geometry — Vulkan Guide. https://vkguide.dev/docs/ascendant/ascendant_geometry/ (retrieved June 2026). Describes the gigabuffer strategy, indirect GPU culling, and the memory vs. detail tradeoff in large chunk counts.

[5] Johnson, L. D. (2022). "How I made a multi-threaded voxel engine in TypeScript." DEV Community. https://dev.to/lucasdamianjohnson/how-i-made-multi-threaded-voxel-engine-in-typescript-1e8f (retrieved June 2026). Illustrates the job-system threading model and the risk of unbounded memory growth when the main thread queues work faster than workers consume it.

[6] Gustafsson, D. (2020). Voxagon Blog — Teardown technical posts. https://blog.voxagon.se/ (retrieved June 2026). Describes Teardown's dense voxel format, palette-based material system, and the entity-component engine architecture.
