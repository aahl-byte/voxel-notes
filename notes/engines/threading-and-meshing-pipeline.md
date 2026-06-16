<link rel="stylesheet" href="./css/globals.css">

# threading and the meshing pipeline

The camera moves. The player sprints toward unexplored terrain. Dozens of new chunks slide
into view range. Each one needs voxel data generated, a mesh built from that data, and vertex
buffers uploaded to the GPU. On modest hardware, generating and meshing a single chunk can take
anywhere from a fraction of a millisecond to tens of milliseconds. Do that work on the thread
that is also driving the render loop, and the frame stalls. The player sees a hitch — a
visible freeze or stutter — every time a chunk comes in. Do it wrong enough times and a 60fps
game drops to a slideshow the moment the player moves.

The goal of the threading and meshing pipeline is simple: <em>keep the render thread free</em>
so frames arrive at a steady cadence, while chunk generation, meshing, and lighting work happens
in the background on other CPU cores. The render thread just draws what is ready; it never waits
for a chunk to be built. Understanding how engines do that — and what can go wrong — is what
this page is about.

Cross-cutting concerns that sit downstream of this pipeline: how chunks are chosen for loading
in the first place is covered in [chunk management and streaming](./chunk-management-and-streaming.md),
and how edits interact with the re-mesh cycle is in [runtime editing and CSG](./runtime-editing-and-csg.md).

---

## the core problem — milliseconds on the wrong thread

Rendering a frame takes roughly 16ms at 60fps. Every millisecond spent on the main thread doing
non-render work is a millisecond stolen from that budget. Chunk work is brutally expensive by
those standards:

- **voxel generation** — running noise functions or reading from disk to fill a chunk's data
  array: 1–20ms depending on complexity and chunk size
- **meshing** — scanning the voxel grid, culling hidden faces, and emitting vertex data:
  0.05ms for binary greedy meshing on a small chunk [1], up to several milliseconds for large
  or dense chunks with expensive algorithms
- **lighting** — flood-filling light through the chunk and its neighbors: comparable to meshing
  or worse
- **GPU upload** — copying vertex buffers across the CPU–GPU bus: typically under a millisecond
  for a single chunk, but it adds up when dozens arrive at once

Put any of that on the render thread and you will see hitches. The solution is to push
everything off the render thread that can be pushed. What remains on the render thread is only:
upload the finished mesh data and draw it. All the building happens elsewhere.

For a map of where this pipeline sits in the broader engine, see
[anatomy of a voxel engine](./anatomy-of-a-voxel-engine.md). For what meshing actually
produces, see [why mesh voxels](../meshing/why-mesh-voxels.md).

---

## the fix — a job system with a priority queue

### the coarse model

Imagine a shared list of work items on a board. Each item is a chunk job: "generate chunk at
(32, 0, 48)", "mesh chunk at (0, 0, 16)", "update lighting for (−32, 0, 0)". A group of worker
threads — one or two fewer than the number of CPU cores, so the render thread always has
headroom — grab items off the board, do the work, and drop the finished result into a results
tray. Once per frame, the render thread glances at the tray, picks up whatever is ready, and
uploads it to the GPU. It never stops to wait.

That arrangement — a shared queue of work items, a pool of threads that pull from it, results
handed back to the caller — is what engineers call a <em>job system</em> or <em>task system</em>.
The shared list is the task queue; the items are jobs; the threads are the thread pool.

### priority — closest chunks first

Not all chunk jobs matter equally. A chunk 2 meters in front of the camera is urgent; one 300
meters away can wait. Engines sort the task queue by distance from the camera, so workers always
pull the most important chunk first. As the player moves, priorities are updated — a chunk that
was far away becomes urgent, and its job moves toward the front.

This is called a <em>priority queue</em>: a queue where items are ordered by importance rather
than arrival time. Godot's voxel module uses priority scheduling across its thread pool so that
nearer chunks preempt farther ones [2].

### the stages of a chunk job

A full chunk lifecycle typically decomposes into at least three sequential jobs — each depending
on the one before, so they chain rather than run in parallel for the same chunk:

1. **generate** — fill the chunk's voxel array with terrain data (noise, SDF, loaded from disk)
2. **mesh** — scan the voxel array and emit vertex data; requires the chunk's six face-neighbors
   to determine which faces are hidden (see [blocky and greedy meshing](../meshing/blocky-and-greedy-meshing.md))
3. **light** — flood-fill light values; like meshing, needs neighbor data

Each stage produces output that feeds the next. Engines often keep these as separate job types
in the queue, so the scheduler can interleave work across many chunks rather than processing one
chunk end-to-end before starting the next. A concrete six-step breakdown — CPU meshing → buffer
creation → CPU-to-upload transfer → GPU copy command → fence check → add to render list — is
documented in [4].

---

## data safety — who owns the chunk while it is being meshed

The most dangerous moment is while a worker thread is reading a chunk's voxel data to build a
mesh. If the main thread simultaneously writes a block edit into that same data, the worker may
read a partially-updated state and produce a corrupt or wrong mesh. This is a data race, and it
produces bugs that are silent, intermittent, and hard to reproduce.

There are two clean ways engines avoid this.

### option 1 — the worker owns the chunk

While a chunk job is in flight, the main thread is not allowed to touch its voxel data. The
chunk is logically "locked" to the worker. Player edits that arrive during this window do not
modify the data in place — they are recorded in a pending-edits list. When the job finishes and
returns the chunk, the edits are applied and the chunk is marked dirty, triggering a re-mesh.

This keeps the worker's view completely stable without copying any data. The cost is that edits
are slightly delayed — they do not take effect until after the current mesh job completes.

### option 2 — an immutable snapshot

The worker receives a read-only copy of the chunk's voxel data — a <em>snapshot</em> — taken at
the moment the job is enqueued. The live data on the main thread can be modified freely while
the worker meshes from the snapshot. When the mesh is done, it reflects the state of the world
at job-enqueue time. If an edit arrived after the snapshot was taken, the chunk is re-queued as
dirty and will be re-meshed again.

The tradeoff is copy cost. For a dense 32³ chunk of 4-byte voxels, a snapshot is 128 KB. That
is manageable, but it multiplies with many concurrent jobs. Some engines use reference-counted
shared pointers to chunk data instead of copying — the worker holds a reference to the old
version, the main thread writes a new version, and the old one is released when the worker is done.

Godot's voxel module uses a `SpatialLock3D` — a mutex-protected list of spatial boxes — to
coordinate access, allowing multiple simultaneous readers but serializing writes [2]. Single-voxel
edits lock the block, apply the change, and immediately re-queue the chunk for re-meshing.

### the dirty flag and re-queue

When an edit modifies a chunk that either has an in-flight mesh job or has already been meshed,
the engine sets a <em>dirty flag</em> on that chunk. A dirty chunk must be re-meshed before its
current mesh is trusted. The usual flow:

- edit arrives → mark chunk dirty → if a job is in flight, let it complete (the result will be
  discarded), then enqueue a new job; if no job is in flight, enqueue immediately
- edits at a chunk's boundary also dirty the face-neighbor chunks, since those neighbors' meshes
  include shared-face visibility data

The dirty flag is typically just a boolean or bitfield per chunk that the main thread checks each
frame before deciding what to enqueue.

---

## double-buffering the mesh

The render thread reads vertex buffer data every frame. The upload step writes new vertex buffer
data when a mesh job completes. These two operations cannot happen at the same moment on the
same buffer — the renderer would read garbage mid-write.

The standard fix is to keep <em>two mesh buffers</em> per chunk: a "front" buffer the renderer
is currently drawing from, and a "back" buffer where the new mesh is written. When the upload is
complete, the front and back swap atomically — a single pointer or index update. On the next
frame, the renderer uses the new front buffer without having seen any in-progress state.

This is <em>double buffering</em> applied to mesh data. The render thread always reads from a
complete, finalized buffer. The upload step always writes to the idle buffer. The swap is the
only moment of coordination needed, and it is fast enough to do under a frame. The same
principle keeps simulation and rendering state independent in the broader engine: frame N is
being rendered while frame N+1 is being computed [8].

The cost is doubled GPU memory for mesh buffers. For a world with thousands of loaded chunks,
this is a real budget item. An alternative is a <em>persistent vertex pool</em>: a single large
buffer pre-allocated at startup, sub-divided into fixed-size buckets, each bucket assigned to
one chunk [3]. New mesh data writes into the chunk's bucket; the old data is overwritten once
the GPU signals — via fence — that the new data is fully uploaded. This trades the
double-memory cost for a fixed pool size but requires careful upfront sizing.

---

## GPU upload — why it happens on the main thread

Worker threads build the mesh in system RAM: they fill a CPU-side buffer with vertex data. That
data then has to cross the CPU–GPU bus into device-local memory, which is where the renderer
reads it. This transfer step goes through a <em>staging buffer</em>.

The pattern in Vulkan (and its equivalent in DirectX 12) [4]:

1. Worker writes mesh vertices into a CPU-accessible staging buffer
2. Main thread records a `CopyResource` command copying staging → device-local buffer
3. That copy command is submitted to the GPU's <em>transfer queue</em> — a hardware copy engine
   that runs independently of the render queue [4]
4. A fence signals when the copy is complete; only then is the chunk added to the render list

In older APIs (OpenGL, DirectX 11), uploads are simpler — `glBufferData` or `UpdateSubresource`
— but they still need to be called from the thread that owns the graphics context, which is
typically the main thread. Workers produce the data; the main thread hands it to the GPU.

### batching uploads

Uploading one chunk per frame wastes the transfer queue — there is budget for more. A common
strategy is to set a per-frame upload budget: upload at most N meshes per frame, or spend at
most T milliseconds on uploads before yielding. Godot's voxel module uses a configurable
main-thread time budget, defaulting to ~8ms per frame, to bound how much finalization work
happens before returning control to the render loop [2]. The vkguide Ascendant engine example
goes further: it pre-allocates a 400MB gigabuffer and uses a compute shader to scatter-write
block data into device-local memory at the start of each frame, avoiding per-chunk upload
commands entirely [5].

---

## pitfalls

### oversubscription

Creating more worker threads than there are CPU cores does not make the pipeline faster — it
makes it slower. Each extra thread competes for the same execution slots, and the OS spends time
context-switching between them. The standard rule: create (number of hardware cores − 1) worker
threads at most, leaving one core for the render thread and OS work. Godot's voxel module
deliberately avoids using all available cores because players run background applications and the
render thread needs uncontested headroom [2].

### false sharing

Modern CPUs cache memory in 64-byte cache lines. If two worker threads both write to data that
happens to sit within the same cache line — even to different bytes — each write invalidates the
other thread's cached copy, forcing constant re-fetching. This is <em>false sharing</em>, and it
can drag parallel performance close to serial speeds with no visible synchronization in the code.

The fix: align job objects and per-thread data structures to 64-byte boundaries so that fields
written by different threads land on different cache lines. Lock-free job queues pad each job
entry to the hardware destructive interference size for exactly this reason [6].

### lock contention

A naive job queue protected by a single mutex becomes a bottleneck when many threads try to push
or pop at the same moment. All workers serialize on the lock and throughput collapses.

The improvement is a <em>lock-free queue</em> — implemented with atomic compare-and-swap
operations rather than a mutex. An even more scalable design is a <em>work-stealing queue</em>:
each thread has its own local queue; when a thread runs out of work it steals from another
thread's queue. The owner thread pushes and pops from one end (LIFO); thieves steal from the
other end (FIFO). This eliminates single-point contention almost entirely [6]. Intel's GTS
(Games Task Scheduler) is a production-grade work-stealing library designed for game engines
and used in shipping titles [7].

### mid-mesh edits and generation invalidation

If the player edits a voxel in a chunk whose generation job has not started yet, the edit will
be wiped out when generation runs — generation overwrites the voxel array from scratch. Engines
handle this by recording edits as a separate layer that is applied on top of generated data.
After generation completes, the edit layer is composited in before meshing begins. This is the
"paint over noise" pattern used in Minecraft-style engines and detailed further in
[runtime editing and CSG](./runtime-editing-and-csg.md).

---

## putting it together — the full frame

One frame of the threading pipeline looks like this.

**main thread, render loop:**
1. Check camera position; compute which chunks need loading or unloading
2. Pop finished mesh results from the results queue; upload staged vertex data to the GPU
3. Swap double buffers for newly uploaded chunks
4. Draw the scene
5. Push new chunk jobs onto the priority queue (generate / mesh / light) ordered by camera distance

**worker threads, running continuously:**
1. Pop the highest-priority job from the shared queue
2. Do the work (generate, mesh, or light) on local data; write nothing that the main thread or
   renderer owns
3. Push the finished result onto the results queue
4. Loop back to step 1

The render thread never blocks on chunk work. Workers never touch GPU resources. The results
queue is the only shared handoff point, and it is kept lock-free.

### producer/consumer sketch

```cpp
// Worker thread — produces finished meshes
while (running) {
    ChunkJob job = jobQueue.pop();           // blocks until work available
    MeshData mesh = buildMesh(job.snapshot); // pure computation, no shared writes
    resultQueue.push({ job.chunkId, mesh }); // hand off to main thread
}

// Main thread — consumes finished meshes, once per frame
while (resultQueue.tryPop(result)) {
    uploadToGPU(result.chunkId, result.mesh); // staging → device-local
    chunkMap[result.chunkId].swapBuffer();    // front ↔ back swap
}
```

### double-buffer swap sketch

```cpp
struct ChunkMesh {
    GPUBuffer buffers[2]; // front [0] and back [1]
    int       front = 0;  // index the renderer reads

    // Called on the main thread after upload to back is confirmed complete
    void swapBuffer() {
        front ^= 1;       // 0→1 or 1→0; atomic in practice
    }

    GPUBuffer& readBuffer()  { return buffers[front]; }
    GPUBuffer& writeBuffer() { return buffers[front ^ 1]; }
};
```

---

## references

[1] cgerikj. "binary-greedy-meshing." GitHub. Binary greedy mesher benchmarks: 50–200µs per
chunk. [source](https://github.com/cgerikj/binary-greedy-meshing)

[2] Gilleron, M. (Zylann). "Performance — Voxel Tools Documentation." Read the Docs. Threading
architecture, spatial lock, main-thread time budget, Vulkan mesh building on threads.
[source](https://voxel-tools.readthedocs.io/en/latest/performance/)

[3] McDonald, N. (2021). "High Performance Voxel Engine: Vertex Pooling." Persistent vertex
pool, bucket architecture, double-buffer avoidance, performance results.
[source](https://nickmcd.me/2021/04/04/high-performance-voxel-engine/)

[4] Ramaswamy, T. (2024). "Multi-Threaded + Async Copy Queue Chunk Loading System." Six-stage
pipeline, staging buffer, copy queue with fence values, worker thread pattern.
[source](https://rtarun9.github.io/blogs/async_copy/)

[5] vkguide.dev. "High-Performance Voxel and Mesh Rendering." 400MB gigabuffer, compute-shader
scatter-write, GPUUnitList upload at frame start.
[source](https://vkguide.dev/docs/ascendant/ascendant_geometry/)

[6] Martinez, M. (2017). "Lock-Free Job Stealing with Modern C++." Work-stealing deque, false-
sharing cache-line padding, acquire-release memory ordering.
[source](https://manu343726.github.io/2017-03-13-lock-free-job-stealing-task-system-with-modern-c/)

[7] GameTechDev. "GTS — Games Task Scheduler." Intel's production work-stealing task scheduler
for game engines.
[source](https://github.com/GameTechDev/GTS-GamesTaskScheduler)

[8] Loggini, R. (2020). "Render Thread Jobification." Double-buffered game/render state,
priority queues, render proxy objects, ideal task duration 500–2000µs.
[source](https://logins.github.io/programming/2020/12/31/RenderThreadJobification.html)
