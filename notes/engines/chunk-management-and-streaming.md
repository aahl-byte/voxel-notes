<link rel="stylesheet" href="./css/globals.css">

# chunk management and streaming

Minecraft's Overworld has no loading screen between biomes. No Man's Sky lets you descend from orbit to ground level without a pause. Neither game ever says "you have run out of world." Both run on consumer hardware with a few gigabytes of RAM, yet their worlds are effectively boundless.

The mechanism behind this is not exotic. It is a disciplined policy for deciding which pieces of the world live in memory right now, which ones are being loaded, and which ones can be safely discarded — all driven by where the camera is and where it is heading. That policy, and the data formats and scheduling machinery that support it, is what this page is about.

---

## the coarse model

The world is divided into a regular grid of fixed-size 3D blocks. Each block is called a <em>chunk</em> — an independently loadable unit of voxel data covering a fixed cubic region of world space. The full world exists on disk (or is regenerated procedurally on demand), but only the chunks within a configurable radius around the camera are ever allocated in RAM.

As the camera moves, the radius moves with it. Chunks that enter the radius are loaded or generated. Chunks that leave it are evicted from memory and, if the player modified them, written back to disk. The world feels infinite because the camera always sits at the center of a sliding window of live data. RAM usage stays bounded because the window has a fixed size.

Three things make this work together:

- **the coordinate system** — a consistent mapping between world positions, chunk identities, and local positions within a chunk
- **the streaming policy** — rules for when to load, when to evict, and how to avoid thrashing
- **the persistence format** — a compact on-disk representation that stores only what cannot be regenerated

The rest of this page builds each of these out.

---

## the world as a grid of chunks

### coordinate math

Every position in the world belongs to exactly one chunk. The conversion is straightforward arithmetic, and getting it right is the foundation everything else rests on. Assume chunks are cubic with side length `C` (commonly 16 or 32 voxels):

```
chunk_x = floor(world_x / C)
chunk_y = floor(world_y / C)
chunk_z = floor(world_z / C)
```

Within a chunk, the local (voxel) coordinates are the remainder:

```
local_x = world_x mod C   (always in [0, C-1])
local_y = world_y mod C
local_z = world_z mod C
```

Going the other direction — from a chunk coordinate and a local coordinate back to world space — is equally direct:

```
world_x = chunk_x * C + local_x
```

These three coordinate spaces — <em>world space</em>, <em>chunk space</em>, and <em>local space</em> — are the vocabulary the rest of the engine speaks. The chunk's identity is its (chunk_x, chunk_y, chunk_z) triplet; it is typically used as a key in a hash map from chunk coordinate to the chunk's data. The details of how that data is laid out inside the chunk are covered in [dense grids and chunks](../storing/dense-grids-and-chunks.md).

### why the chunk size matters

The choice of `C` controls a four-way tradeoff:

| smaller `C` | larger `C` |
|---|---|
| finer streaming granularity — a smaller region loads or evicts at once | fewer hash-map lookups per world query |
| more chunk objects to manage | each chunk is a larger allocation |
| cheaper to regenerate mesh for one chunk | longer time to remesh on modification |
| better cache fit during traversal | better compression within one chunk (longer uniform runs) |

Minecraft uses `C = 16` (each section is 16×16×16 voxels) [1]. Many indie engines and the Zeux terrain engine use `C = 32` [2]. Values above 64 are rare in games because mesh rebuild time grows with the cube of `C` and becomes noticeable.

---

## the load/evict policy — the streaming ring

### the basic idea

Keep a sphere (or, in practice, a cube in chunk-space) of chunks resident around the camera. Call its radius `R` chunks. When the camera moves and a chunk enters this sphere, schedule it for loading. When a chunk leaves, schedule it for eviction.

That is the complete coarse model. Everything below is refinement.

### hysteresis — two radii, not one

Using a single radius for both load and evict causes <em>thrashing</em>: a chunk on the boundary gets loaded, then the camera nudges away and it gets evicted, then the camera nudges back and it gets loaded again, in a tight loop. The fix is to separate the two thresholds:

- **load radius `R_load`**: chunks within this distance get scheduled for loading
- **evict radius `R_evict`**: chunks beyond this distance get scheduled for eviction, where `R_evict > R_load`

The gap between the two radii is the <em>hysteresis band</em>. A chunk that was loaded stays resident until it is clearly outside `R_evict`, not until it barely exits `R_load`. One real implementation uses a load radius of 8 chunks and an evict radius of 12 chunks [3] — a 4-chunk hysteresis band.

### the working set

The collection of chunks currently resident in memory is the <em>working set</em>. Its size is bounded: a spherical load radius of `R` chunks contains roughly `(4/3)π R³` chunks. At `R = 8` with `C = 16`, that is about 2,145 chunks, each holding 4,096 voxels at 1 byte each — roughly 8 MB of raw voxel data, well within budget [1]. RAM use scales with `R³`, so doubling the render distance multiplies the working set by 8.

For the GPU, the budget is stricter. The Aokana GPU-driven voxel framework reports loading only approximately 5% of total scene data into VRAM at any time, with peak VRAM usage of 424 MB against 23 GB of total scene data — a roughly 55× reduction achieved through on-demand streaming [4].

### prioritizing the load queue

Not all chunks in the load radius are equally urgent. A chunk directly in front of the camera should finish loading before one behind it. The standard approach is to sort the load queue by distance to the camera, closest first. The Voxel Plugin uses a 64-bit priority value combining a category and a distance term:

```
priority = (PriorityCategory << 32) | (TaskPriority + PriorityOffset)
```

Within one category, lower distance = higher priority. Between categories, render-data tasks outrank collision cooking, which outranks meshing, which outranks foliage and decoration [5].

### chunk states

A real engine tracks each chunk through a small state machine. A common set of states:

- **unloaded** — not in memory; not scheduled
- **queued for load** — in the load queue, waiting for a worker thread
- **loading / generating** — a worker thread is computing voxel data
- **loaded** — voxel data is resident; mesh not yet built
- **meshing** — a worker thread is building the draw mesh
- **ready** — voxel data and mesh are both resident; chunk can render
- **queued for evict** — outside `R_evict`; will be freed after optional disk write

The update loop runs once per frame, processes each queue in bounded batches (e.g., up to 32 chunks per frame per queue [6]), and transitions chunks between states. Processing in bounded batches prevents any single frame from spending unbounded time on chunk I/O while the rest of the game stalls.

The visibility pass only recomputes which chunks are in the render ring when the camera crosses a chunk boundary — not every frame — keeping the overhead negligible on frames where the player stands still [3].

---

## persistence — storing only what changed

### procedural worlds are not fully stored

A world large enough that no player will ever visit most of it cannot be fully stored on disk. Minecraft's world is theoretically 60 million × 60 million blocks; storing the whole thing would require petabytes. The solution is to store only what procedural generation cannot reproduce.

For any unmodified chunk, the engine can re-derive the terrain from the seed and position alone. The disk stores only the <em>delta</em>: the set of voxel changes a player made that differ from what the procedural generator would produce [7]. When a chunk is loaded, the engine generates the base terrain, then applies the stored delta on top.

Once a chunk has been modified, most engines abandon the delta model and write the full chunk to disk. Tracking fine-grained deltas is complex to implement correctly (especially under multiple editing sessions), and a compressed full chunk is small enough that the simplicity is worth it.

### region files

Storing one file per chunk produces millions of tiny files — the filesystem struggles with directory traversal, and the OS metadata overhead per file becomes significant. The standard solution, pioneered by Minecraft's MCRegion format (later superseded by Anvil), groups 32×32 = 1,024 chunks into a single <em>region file</em> [1].

The region file structure:

- **8 KiB header** (two sectors of 4 KiB each):
  - Sector 0: 1,024 chunk location entries (4 bytes each) — each entry encodes the sector offset (3 bytes) and the sector count (1 byte) of that chunk's compressed data within the file
  - Sector 1: 1,024 timestamps (4 bytes each) — the last-write time of each chunk
- **data region**: chunk payloads packed end-to-end, each preceded by a 5-byte header (4-byte length + 1-byte compression type), padded to a multiple of 4 KiB

The region file name encodes the region's (X, Z) position: `r.X.Z.mca`. To find which region contains chunk (cx, cz): region_x = floor(cx / 32), region_z = floor(cz / 32). To find the chunk's slot within that region: index = (cx mod 32) + 32 × (cz mod 32) [1].

### compression

Each chunk's NBT-encoded data is compressed before writing. Minecraft uses zlib (RFC 1950) by default, with LZ4 available since snapshot 24w04a as a faster alternative [1]. The Zeux engine reports approximately 0.07 bytes per voxel for RLE-compressed terrain, falling to around 0.04 bytes per voxel after an additional LZ4 or zstd pass [2]. For context, an uncompressed 16³ chunk at 2 bytes per voxel costs about 8 KB; the same chunk compressed typically fits in 200–500 bytes for typical terrain.

The Anvil format made one important change from MCRegion: it reordered block data from XZY to YZX column order. Terrain tends to have long uniform horizontal layers (stone, air), so making Y vary slowly (and thus sorting blocks by column) produces longer runs of identical values — the same data compresses better [1].

For a deeper look at the compression algorithms used within and across chunks, see [compression techniques](../optimization/compression-techniques.md).

### what gets stored besides voxels

A region file chunk contains more than raw block data:

- **block palette** — the mapping from short palette indices to block type IDs (see [dense grids and chunks](../storing/dense-grids-and-chunks.md))
- **biome array** — one biome ID per column, stored explicitly so biomes survive world-generator updates without mis-generating
- **heightmap** — precomputed per-column surface heights for fast raycast and lighting
- **entity and tile-entity data** — chests, furnaces, mob spawners and their state
- **lighting data** — cached sky and block light values to avoid recomputing on load

---

## precision at planet scale — floating-point origin rebasing

### what goes wrong far from the origin

Virtually all real-time 3D engines represent positions as 32-bit IEEE 754 floating-point numbers. A 32-bit float has about 7 decimal digits of precision total, and that precision is not evenly distributed — it is concentrated near zero and halves for every doubling of distance from the origin.

At 16,777,216 meters (2²³) from the origin, the smallest representable increment in a 32-bit float is exactly 1 meter. Sub-meter positions cannot be expressed at all. At ±1,048,576 blocks, Minecraft's block corners — which must be multiples of 1/16 — can no longer be represented accurately, causing rendering glitches [8]. Bedrock Edition, which uses 32-bit floats for player position, exhibits severe jitter beyond this range [9]. Java Edition uses 64-bit doubles for most calculations, pushing the threshold to 2⁵³ before analog problems appear.

For a planet-scale or space game (Elite Dangerous, Kerbal Space Program, No Man's Sky), 32-bit coordinates fail catastrophically: vertices snap to coarse grid positions, physics joints break because velocity precision fails, and collisions misbehave.

### the fix: floating-point origin rebasing

The solution is to keep the camera at or near the world-space origin at all times, and instead move the world around the camera [10]. The camera's true position is tracked in 64-bit double precision. Whenever the camera drifts more than some threshold from the origin (commonly 10,000–50,000 units), the engine <em>rebases</em>: it snaps the camera back to the origin and translates every scene object by the negative of that offset. From the player's perspective, nothing changes. From the engine's perspective, every rendered position is now within a bubble of high-precision space near the origin.

The rebase procedure:
1. Record the camera's current 64-bit world position `P`.
2. Shift everything in the scene by `−P` (in 64-bit arithmetic).
3. Set the camera's position to (0, 0, 0).
4. All subsequent rendering and physics happen in this recentered space.

Physics requires additional care: velocity vectors must also be rebased, and in rotating reference frames (a planet surface) fictitious forces (Coriolis, centrifugal) must be accounted for to keep simulation correct [10].

Chunk coordinate math is unaffected by rebasing because chunk coordinates are integers computed in 64-bit space; only the render-time conversion from world to camera-local coordinates is sensitive to floating-point error.

---

## where the work runs — generation, meshing, and the job pipeline

Chunk generation and meshing are the two most expensive operations in a streaming engine. Neither belongs on the main thread.

### separating the stages

A chunk passes through two heavy computation stages before it can render:

1. **Generation (voxel data)** — computing which block type fills each cell. For procedural terrain, this involves noise evaluation, biome logic, decoration, and cave carving. This is pure computation: no GPU, no rendering state, easily parallelized.

2. **Meshing (draw geometry)** — running surface extraction over the voxel data to produce the triangle mesh that the GPU will render. This requires the voxel data of the chunk and its six face-adjacent neighbors (to handle cross-chunk faces correctly). See [threading and meshing pipeline](./threading-and-meshing-pipeline.md) for the surface extraction algorithms.

These two stages are independent per chunk and can run concurrently across many worker threads. The main thread remains free for input, camera updates, and rendering.

### scheduling by distance

The job queue prioritizes by proximity. Chunks closest to the camera get generated and meshed first; distant chunks wait. This produces a smooth experience: the visible region is always fully populated, while chunks being loaded for future movement are still in progress at the edges.

A common implementation uses a thread pool with a priority queue. The Voxel Plugin's 64-bit priority formula places higher-priority categories (collision, meshing) ahead of lower ones (foliage, decoration), and within each category, sorts by camera distance [5]. Some systems cap the number of tasks dispatched per frame — for example, up to 32 chunk copies per frame — to avoid GPU upload stalls [6].

No Man's Sky extends this to a fully continuous pipeline: terrain, polygonization, texturing, population, and simulation all run as overlapping stages so there is never a hard pause waiting for a single stage to complete [11].

### LOD and the streaming hierarchy

At long range, full-resolution chunks are unnecessary and expensive. [LOD in engines](./lod-in-engines.md) covers the full detail — the short version here is that the streaming ring often has multiple concentric zones. The inner zone carries full-resolution chunks. Outer zones carry progressively coarser representations (halved resolution per LOD level). The Aokana framework implements this as an implicit octree: each LOD level's chunk covers a region 2× larger in each axis, so eight LOD-0 chunks aggregate into one LOD-1 chunk at the same 256³ resolution [4].

The streaming decision — which LOD to load for a given chunk — is made per-frame from a formula that compares projected chunk size against a quality threshold:

```
LOD_error = (ChunkSize × StreamingFactor) − distance(ChunkCenter, CameraPos)
```

When `LOD_error > 0`, the chunk is close enough to warrant loading at that LOD level [4]. Adjusting `StreamingFactor` globally scales quality versus memory use.

For the procedural generation that populates chunks with terrain, biomes, and structures, see [procedural terrain](../generating/procedural-terrain.md). For the [anatomy of a voxel engine](./anatomy-of-a-voxel-engine.md), including how all these subsystems wire together into a running game, that page is the high-level map.

---

## the specifics

### choosing `R` and the memory budget

At chunk size 16 and load radius 8 chunks (Minecraft's default view distance):
- Chunks in sphere ≈ (4/3)π × 8³ ≈ 2,145 chunks
- Raw voxel data ≈ 2,145 × 4,096 bytes ≈ 8.8 MB (at 1 byte per voxel)
- With chunk meshes (variable, roughly 5–50 KB per chunk): add 10–100 MB
- A view distance of 16 chunks (double) multiplies the chunk count by ~8: ≈ 70 MB raw

The mesh budget dominates the voxel data budget at typical view distances. Memory profiling on a running engine typically shows the mesh pool as the largest consumer, not the voxel arrays.

### the region file lookup

To read chunk (cx, cz) from disk:
1. Compute region: rx = cx >> 5, rz = cz >> 5
2. Open `r.rx.rz.mca`
3. Compute header index: i = (cx & 31) + 32 × (cz & 31)
4. Read bytes 4i to 4i+3 from the file header: upper 3 bytes = sector offset, lowest byte = sector count
5. Seek to sector_offset × 4096 bytes
6. Read the 5-byte chunk header: bytes 0–3 = payload length, byte 4 = compression type (2 = zlib, 4 = LZ4)
7. Decompress payload_length − 1 bytes starting at byte 5

The total I/O is: one file open, one seek, one read of at most 255 × 4,096 = 1,044,480 bytes (the maximum chunk size in this format) [1].

### empty-chunk optimization

Before allocating a chunk and generating its voxel data, the engine can query the generator for the range of density values in that region. If the range is guaranteed to be entirely air (above the terrain ceiling) or entirely solid (below the bedrock floor), the chunk can be represented as a single constant value — no array allocated, no mesh generated. Minecraft implements this: fully air sections are not stored on disk and are not allocated in memory [1]. The Voxel Plugin calls this "empty chunk skipping" and implements it via range analysis of the generator's outputs [5].

### cross-chunk neighbor access

When meshing or simulating a chunk, voxels on the chunk boundary need to read data from adjacent chunks. The standard pattern is to cache pointers to all six face-adjacent chunks at mesh-build time. If a neighbor is not yet loaded, the engine either waits (stalling the mesh until the neighbor arrives) or meshes without the boundary faces and marks the chunk for re-mesh when the neighbor loads. Most engines choose the latter — it avoids blocking the mesh thread — and accept that boundary faces may briefly flicker into view as neighbors load.

---

## references

[1] Mojang. (2024). "Region file format." *Minecraft Wiki*. [source](https://minecraft.wiki/w/Region_file_format) — canonical specification of the MCR/Anvil region file structure, sector layout, compression schemes, and chunk offset table.

[2] Arbuckle, A. (2017). "Voxel Terrain Storage." *zeux.io*. [source](https://zeux.io/2017/03/27/voxel-terrain-storage/) — detailed engineering analysis of a 32³ chunk-based terrain engine with RLE and LZ4 compression measurements (0.07 → 0.04 bytes/voxel).

[3] Iain. (2012). "An Analysis of Minecraft-like Engines." *0fps.net*. [source](https://0fps.net/2012/01/14/an-analysis-of-minecraft-like-engines/) — benchmark comparison of voxel store strategies; documents the load-at-8 / evict-at-12 radius pattern and its role in avoiding chunk thrashing.

[4] Fang, Y., Wang, Q., and Wang, W. (2025). "Aokana: A GPU-Driven Voxel Rendering Framework for Open World Games." *Proceedings of the ACM on Computer Graphics and Interactive Techniques*. DOI: 10.1145/3728299. [local PDF](../papers/fang-2025-aokana-gpu-driven-voxel-rendering.pdf) · [source](https://arxiv.org/html/2505.02017v1) — describes the LOD error formula, octree-based streaming with ~5% VRAM utilization, and the 55× disk-to-VRAM ratio achieved at 64K resolution.

[5] Hollander, A. (2023). "Performance and Profiling." *Voxel Plugin Documentation*. [source](https://docs.voxelplugin.com/1.2/technical-notes/performance-and-profiling) — documents the 64-bit task priority formula `(PriorityCategory << 32) | (TaskPriority + PriorityOffset)`, empty-chunk skipping via range analysis, and spatial locking for parallel world editing.

[6] Ramaswamy, T. (2023). "Multi-Threaded + Async Copy Queue Chunk Loading System." *rtarun9.github.io*. [source](https://rtarun9.github.io/blogs/async_copy/) — documents a GPU copy queue architecture that caps chunk transfers at 32 per frame using CPU-side fence tracking.

[7] Procedural Worlds contributors. (2013). "Storage Matters." *procworld.blogspot.com*. [source](https://procworld.blogspot.com/2013/03/storage-matters.html) — explains the delta persistence model: store only what the procedural generator cannot reproduce; modified chunks are serialized in full.

[8] Mojang. (2024). "Java Edition distance effects." *Minecraft Wiki*. [source](https://minecraft.wiki/w/Java_Edition_distance_effects) — documents the floating-point precision thresholds: rendering glitches at X/Z ±1,048,576 (2²⁰), lighting breakdown at X/Z ±33,554,432 (2²⁵), crash at 2³¹−1.

[9] Mojang. (2024). "Bedrock Edition distance effects." *Minecraft Wiki*. [source](https://minecraft.wiki/w/Bedrock_Edition_distance_effects) — documents severe movement jitter and fall-through bugs in Bedrock's 32-bit float coordinate system beyond ±16,777,216 blocks (2²⁴).

[10] Rouquier, G. (2023). "Spatial Rebasing: An Unreal Engine Odyssey." *gamedevtricks.com*. [source](https://gamedevtricks.com/post/origin-rebasing-space/) — explains the rebase algorithm, velocity rebasing, and orbital fictitious-force corrections; demonstrates that 32-bit floats lose sub-meter precision at 2²³ meters and sub-centimeter precision far sooner.

[11] McKendrick, I. (2017). "Continuous World Generation in No Man's Sky." *GDC Vault*. [source](https://www.gdcvault.com/play/1024265/Continuous-World-Generation-in-No) — describes the overlapping voxel generation, polygonization, texturing, and population pipeline that eliminates discrete load pauses during planetary traversal.

[12] Crassin, C., Neyret, F., Lefebvre, S., and Eisemann, E. (2009). "GigaVoxels: Ray-Guided Streaming for Efficient and Detailed Voxel Rendering." *Proceedings of ACM I3D 2009*. DOI: 10.1145/1507149.1507152. [local PDF](../papers/crassin-2009-gigavoxels-ray-guided-streaming.pdf) · [source](https://maverick.inria.fr/Publications/2009/CNLE09/) — introduces the GPU brick pool with LRU eviction and ray-guided demand loading: only bricks that a ray actually traverses are requested, tightly coupling streaming demand to the rendered image. Brick size M = 32³ voxels; renders billions of voxels at 20–90 fps within a fixed GPU memory budget.
