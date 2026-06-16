<link rel="stylesheet" href="./css/globals.css">

# dense grids and chunks

Minecraft's Overworld is effectively infinite. Players load into it, dig hundreds of meters down, build sprawling structures, and travel thousands of blocks in any direction — all at real-time frame rates, on consumer hardware. The underlying store that makes this possible is not exotic: it is fixed-size 3D blocks of data held in flat arrays, loaded and unloaded as the player moves through the world. Everything more sophisticated — sparse trees, hash grids, VDB — is measured against this baseline.

This page builds up that baseline from the ground, then shows the two in-chunk compression techniques that keep it affordable: run-length encoding and palette compression. The tradeoffs it pays will point directly to when you should switch to a [sparse structure](./octrees-and-svo.md).

---

## the coarse model

A region of the world is divided into a regular grid of cells. Those cells are stored in a flat, one-dimensional array in memory. The array is indexed by a single integer computed from the cell's (x, y, z) coordinates. You want the value at a given position? One arithmetic expression, one array lookup. That is the entire read path.

The world is too large to hold in memory all at once, so it is divided into fixed-size 3D blocks — call them <em>chunks</em>. Each chunk is its own flat array. Only the chunks near the player are allocated; distant chunks are evicted and, if modified, written to disk. The world becomes as large as storage allows, not as large as RAM allows.

Within each chunk, most cells are typically the same thing — air, stone, or water. Two techniques exploit this uniformity to shrink the array without losing random-access ability: <em>run-length encoding</em>, which stores runs of identical values as a single (value, count) pair, and <em>palette compression</em>, which stores a small lookup table of distinct block types and replaces each cell's full type ID with a short index into that table.

A beginner can stop here. Flat array + chunking + compression is the architecture of Minecraft and most block games. The rest of this page is the mechanism behind each of those three words.

---

## the flat 3D array

The foundation of voxel storage is established in [the voxel grid](../foundations/the-voxel-grid.md): a rectangular region of space divided into W × H × D cells, each holding one value. In memory, a 3D array is just a 1D array of length W × H × D. The mapping from a 3D coordinate to an array position is the <em>linear index</em>:

```
index = x + y * W + z * W * H
```

`W` is the width (cells along x), `H` is the height (cells along y). As `x` increments by 1, the index increments by 1 — stepping along x is stepping through consecutive memory addresses. Walking along z jumps by `W * H` elements, which can span many cache lines on a large grid.

### what this buys you

- **O(1) random access.** Given any (x, y, z), the index is three multiplies and two adds. No tree traversal, no hash lookup, no pointer chasing.
- **Excellent sequential performance.** Sweeping the grid in x-major order is a sequential read of a flat buffer — the fastest possible memory access pattern for modern CPUs [1].
- **Trivial neighbor lookup.** The six face-adjacent neighbors of any cell at index `i` are at `i ± 1` (x-neighbors), `i ± W` (y-neighbors), and `i ± W*H` (z-neighbors). No adjacency list, no graph traversal.

### what it costs

Memory scales with the cube of linear resolution. A 256 × 256 × 256 grid of 4-byte integers costs 64 MB. Double the resolution in each axis and that becomes 512 MB. This is the [storage problem](./the-storage-problem.md) in its most direct form: most voxel grids are mostly empty, so a flat array wastes memory on cells that hold nothing interesting [2].

Chunking addresses the infinite-world part of that problem. Compression addresses the empty-space part.

---

## chunking — splitting the world into fixed blocks

A single flat array that covers the entire world cannot work: even a modest 4096 × 256 × 4096 world at 1 byte per voxel costs 4 GB, and it must all be allocated at once. The solution is to divide the world coordinate space into a regular grid of equal-sized sub-volumes and give each one its own flat array. That sub-volume is the <em>chunk</em>.

Minecraft uses 16 × 16 × 16 block sections stacked vertically, with up to 24 sections per column (the Overworld extends from y = −64 to y = 320). A section not fully loaded is simply not allocated. The game keeps only the sections near the player in memory [3].

A chunk-sized flat array for a 16³ section costs 4,096 entries — small enough to fit comfortably in L2 cache on most CPUs. This is not an accident.

### why chunk size matters

Chunk size controls a four-way tradeoff:

| smaller chunks | larger chunks |
|---|---|
| finer streaming granularity | fewer hash/map lookups |
| more chunks to manage | each chunk is a bigger allocation |
| better cache fit per chunk | better compression (more uniform runs) |
| cheaper to regenerate mesh | costlier to regenerate mesh |

Common sizes are 16³ (Minecraft), 32³ (used by many indie engines and the Zeux terrain engine [1]), and 64³. There is no universally correct answer; 16³ and 32³ are the most widespread in game engines.

### what chunks enable

- **Streaming.** Load chunks near the player; evict distant ones. The world is unbounded without ever holding it all in memory.
- **Meshing units.** Surface extraction ([chunk management and streaming](../engines/chunk-management-and-streaming.md)) regenerates the visible mesh for one chunk at a time. A chunk is exactly the right granularity: big enough to batch draw calls, small enough to regenerate quickly on a background thread.
- **Independent modification.** Edits to one chunk never invalidate another chunk's data. Only the mesh of the affected chunk (and its edge-adjacent neighbors) needs to be rebuilt.
- **Level of detail.** Distant chunks can store lower-resolution versions of the data. Chunks are a natural LOD boundary.

---

## shrinking the chunk — in-chunk compression

The flat array allocates one entry per cell regardless of content. A chunk of 4,096 cells where 3,800 are air pays for all 4,096. Two compression schemes fix this without giving up the O(1) random access that makes the flat array useful.

### run-length encoding

The cells in a chunk's 1D array often have long stretches of the same value: a thick layer of stone, a large pocket of air, a deep water column. Instead of storing every cell individually, you store a sequence of (value, count) pairs — one pair per run of identical values. That sequence is the <em>run-length encoded</em> representation.

Example: a column of 32 cells — 12 stone, then 15 air, then 5 water — becomes three pairs instead of 32 individual entries:

```
[(stone, 12), (air, 15), (water, 5)]
```

If typical terrain has long uniform layers, compression ratios are dramatic. The Zeux engine reports approximately 0.07 bytes per voxel for RLE-compressed terrain (versus 2 bytes per voxel uncompressed) — roughly 28× compression on disk [1]. Compressing RLE output further with LZ4 or zstd pushes this to around 0.04–0.05 bytes per voxel.

RLE shines for **serialization** (saving chunks to disk, sending them over a network). It is less useful as a primary in-memory format, because looking up a single arbitrary cell requires scanning the run list — O(r) in the number of runs — rather than an O(1) array offset. The 0fps analysis of Minecraft-like engines showed that flat array random read is 0.224 μs while an interval-tree (RLE-like) store is 0.571 μs for the same operation [4].

#### when to reach for RLE

- Disk serialization and network transmission (it is Minecraft's actual on-disk format, combined with zlib).
- Read-heavy workloads dominated by sequential iteration rather than random access — mesh generation sweeping rows of air, for instance.
- Chunks where most content is uniform (underground, outer-space, or empty sky sections).

#### when RLE loses

- High-entropy chunks (dense mixed terrain, player-built structures with many block types in no particular order) can actually *expand* with naive RLE — every run is length 1, so you store a count alongside every value.
- Any workload requiring many random writes interspersed with reads, since each write may split a run.

For the general case see [compression techniques](../optimization/compression-techniques.md), which covers RLE alongside the heavier algorithms applied to voxel data.

### palette compression

A chunk of 16 × 16 × 16 = 4,096 cells drawn from a game with thousands of block types rarely uses more than a handful of distinct types in any one section. Even a complex player build typically uses fewer than 64 distinct materials per chunk. The number of distinct types present is far smaller than the number of possible types in the registry.

Palette compression exploits this. Instead of storing a full block-type ID (which might be 15 bits to cover thousands of registered blocks) per cell, you store two things:

1. A small lookup table — the <em>palette</em> — that lists just the distinct block types present in this chunk. Entry 0 might be air, entry 1 stone, entry 2 oak planks, and so on.
2. A bit-packed array of short indices into that palette — one index per cell.

The index only needs enough bits to address the palette. With 4 or fewer distinct types, 2 bits suffice. With up to 16 types, 4 bits suffice. With up to 256 types, 8 bits. Minecraft's implementation uses a minimum of 4 bits per index and a maximum of 8 before switching to direct (global) IDs; the bit depth is set to `ceil(log₂(palette size))`, rounded up to the nearest valid step [3].

A concrete example: a 16³ section of mostly stone with a few ores and a pocket of air — say 5 distinct block types. The palette holds 5 entries. Each cell needs only 3 bits to index a palette of up to 8 entries. The bit-packed array is `4096 × 3 bits = 12,288 bits = 1,536 bytes`. A 16-bit block ID array of the same section would cost `4096 × 16 bits = 8,192 bytes`. That is a 5× reduction without any change to the data's random accessibility — looking up cell (x, y, z) is still index arithmetic, one palette lookup [5].

```
# pseudocode — write a block
def set_block(chunk, x, y, z, block_type):
    if block_type not in chunk.palette:
        chunk.palette.append(block_type)
    idx = chunk.palette.index(block_type)
    bit_offset = (x + y * W + z * W * H) * chunk.bits_per_index
    pack_bits(chunk.data, bit_offset, chunk.bits_per_index, idx)

# pseudocode — read a block
def get_block(chunk, x, y, z):
    bit_offset = (x + y * W + z * W * H) * chunk.bits_per_index
    idx = unpack_bits(chunk.data, bit_offset, chunk.bits_per_index)
    return chunk.palette[idx]
```

When a section contains only one block type (a solid stone layer, a completely empty air section), the palette has one entry and the bit array can be omitted entirely — the entire section is described by that single entry [3].

#### direct vs indirect palette

The distinction matters at scale:

- **Indirect palette** — the palette is local to the chunk section. Indices point into that local list. This is the efficient path for typical terrain (few distinct types per section).
- **Direct palette** — when the section contains more distinct types than the indirect palette can efficiently represent (more than ~256 in Minecraft's scheme), indices point directly into the global block registry. Random access remains O(1) but the compression benefit shrinks because indices are now as wide as global IDs.

Minecraft transitions from indirect to direct when a section would need more than 8 bits per index. At that threshold, using the global ID directly costs the same per-cell storage and avoids the overhead of maintaining a local palette [3].

#### when palette compression wins

- Typical game-world terrain: stone, air, and a handful of ores per section. Most chunks use 4–16 distinct types, fitting easily in 4 bits.
- Any chunk with large uniform regions — all-air sections above ground, all-stone sections deep underground — compress to near-zero with the single-entry optimization.
- Systems that need O(1) random access (unlike RLE) while still saving memory vs. a raw array.

---

## the tradeoffs — what dense + chunks pays for

The flat-array-plus-chunks approach is the simplest store that actually ships at Minecraft scale. It earns that title with real strengths:

- **Predictable performance.** Read, write, and neighbor access are all constant time with small constants. No allocation surprises, no tree rebalancing.
- **Cache friendliness.** A 16³ or 32³ chunk fits in L2 cache. Iterating all cells is a sequential sweep of a contiguous buffer. Mesh generation — the most iteration-heavy workload — is as fast as memory allows.
- **Simplicity.** Implementation is a flat array, a linear index formula, and an optional palette. A working prototype fits in a few dozen lines.

But it pays real costs:

- **Empty space.** A chunk allocated for even one non-air block pays for all 4,096 cells. A world with thin terrain floating above vast empty space (skyblock maps, space games, cavernous worlds) wastes most of every chunk. Palette compression helps within allocated chunks; chunk culling (never allocating all-air chunks) helps at the chunk level. But regions that are mostly empty are fundamentally inefficient in this scheme.
- **Fixed resolution.** Every cell in a chunk is the same size. You cannot represent distant terrain at lower resolution in the same structure without a separate LOD system.
- **No built-in skipping.** Rendering and ray tracing cannot skip empty space without additional acceleration data — a bounding volume hierarchy, a min/max pyramid, or a separate empty-chunk bitmask. Dense arrays store no information about what to skip.

These are exactly the gaps that sparse structures fill. An [octree or SVO](./octrees-and-svo.md) allocates nodes only where data exists and naturally supports hierarchical skipping — at the cost of O(log n) access and more complex traversal. The choice between them is the subject of [choosing a voxel store](./choosing-a-voxel-store.md).

For a comparison of the memory layout options available within the dense approach — including Morton (Z-order) encoding that improves cache behavior for 3D neighbor access — see [memory layout and Morton curves](../optimization/memory-layout-and-morton.md).

---

## specifics

### the linear index and axis order

The formula `x + y * W + z * W * H` is x-major: x varies fastest (consecutive x values are adjacent in memory), z varies slowest (consecutive z values are `W * H` apart). Sweeping in x is a sequential access; sweeping in z is a strided access with stride `W * H`.

Some engines use z-major order instead (`z + y * D + x * D * H`), making z the fast axis. This is purely a convention choice — what matters is that one convention is picked and applied everywhere. Mixing conventions silently produces wrong neighbor offsets, which typically manifests as terrain that appears shifted or mirrored [4].

The neighbor offset table from [the voxel grid](../foundations/the-voxel-grid.md) is:

| neighbor | index offset |
|---|---|
| ±x | ±1 |
| ±y | ±W |
| ±z | ±W × H |

These offsets hold only for cells that are not on a chunk boundary. Cross-chunk neighbor lookups require a separate mechanism — typically a per-chunk neighbor pointer or a coordinated lookup into the chunk map.

### bit-packing palette indices into 64-bit integers

Palette indices are packed into arrays of 64-bit integers (longs) with no index crossing a word boundary. If the bit width does not divide 64 evenly, the remaining bits of each long are left as padding. With 5 bits per index, for instance, a long holds 12 indices (60 bits used, 4 bits padding). This wastes some space but eliminates the need for slow cross-word bit reads [3].

### the cost of a chunk rebuild

When any block in a chunk changes, the mesh for that chunk (and potentially its neighbors, since faces on the shared boundary may appear or disappear) must be regenerated. Mesh generation is O(n) in the number of cells — fast for 16³ chunks on a background thread, but potentially a bottleneck if chunk size is too large or if many chunks change per frame. Keeping chunk size bounded (≤ 32³ is common) keeps rebuild times predictable. See [chunk management and streaming](../engines/chunk-management-and-streaming.md) for how engines schedule these rebuilds.

### choosing not to allocate empty chunks

The cheapest optimization in a sparse world is to not allocate chunks at all when they are entirely uniform. A hash map from chunk coordinate to chunk pointer, where the absence of an entry implies a known default value (usually air), avoids allocating megabytes for empty sky or empty bedrock. Minecraft implements this: sections filled entirely with air are not written to disk and are not allocated in memory [3]. This is a coarse form of the sparsity that octrees apply at every level of the hierarchy.

---

## references

[1] Arbuckle, A. (2017). "Voxel Terrain Storage." *zeux.io*. https://zeux.io/2017/03/27/voxel-terrain-storage/ — detailed engineering analysis of a 32³ chunk-based voxel terrain engine with RLE compression measurements.

[2] Arbore, R., Liu, J., Wefel, A., Gao, S., and Shaffer, E. (2024). "Hybrid Voxel Formats for Efficient Ray Tracing." *arXiv preprint*. DOI: 10.48550/arXiv.2410.14128. [local PDF](../papers/arbore-2024-hybrid-voxel-formats-ray-tracing.pdf) · [source](https://arxiv.org/abs/2410.14128) — establishes that "memory consumption of uncompressed voxel volumes scales cubically with resolution" and evaluates hybrid hierarchical formats against dense baselines.

[3] Mojang. (2024). "Chunk Format." *Minecraft Wiki*. https://minecraft.wiki/w/Chunk_format — canonical documentation of Minecraft's 16³ section structure, indirect/direct palette scheme, and 4–8 bit index encoding.

[4] Iain. (2012). "An Analysis of Minecraft-like Engines." *0fps.net*. https://0fps.net/2012/01/14/an-analysis-of-minecraft-like-engines/ — benchmark comparison of flat arrays, virtual arrays, octrees, and interval trees; random access timings and iteration analysis.

[5] longor. (2019). "Voxel Palette Compression." *longor.net*. https://www.longor.net/articles/voxel-palette-compression-reddit — detailed implementation guide for per-chunk palette compression with bit-depth calculation.

[6] Fang, Y., Wang, Q., and Wang, W. (2025). "Aokana: A GPU-Driven Voxel Rendering Framework for Open World Games." *arXiv preprint*. DOI: 10.48550/arXiv.2505.02017. [local PDF](../papers/fang-2025-aokana-gpu-driven-voxel-rendering.pdf) · [source](https://arxiv.org/abs/2505.02017) — addresses "the high storage cost of voxels" and demonstrates that sparse SVDAG achieves up to 9× memory reduction over dense representations.
