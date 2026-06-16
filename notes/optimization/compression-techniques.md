<link rel="stylesheet" href="./css/globals.css">

# compression techniques

You want a voxel world that fits in VRAM. Or a save file that fits on a phone. Or a billion-voxel scene that loads at interactive frame rates. In every case, the obstacle is the same: a high-resolution voxel volume in its natural form is enormous, and the hardware budget is fixed. Compression is what makes high resolution affordable — the gap between "doesn't fit" and "runs in real time" is almost always a compression decision.

This page maps the toolbox: what each technique actually does, what it costs to use, and how to pick the right one for your situation. Cross-cutting the whole discussion is one tension that never goes away — making data smaller usually makes it slower to access. Getting that tradeoff right is the real skill.

---

## the coarse model

Before any compression technique is applied, you're paying for every cell in the grid whether it has interesting content or not — the scale of that waste is covered in [the storage problem](../storing/the-storage-problem.md). Compression works by finding and removing one of three kinds of redundancy:

- **Repetition within a region.** Voxel volumes have long runs of the same value — stone, air, water — and blocks with only a handful of distinct materials in any given area. Two compact techniques (RLE and palette indexing) eliminate this redundancy while keeping data accessible.
- **Identical regions across the whole volume.** Real scenes contain enormous amounts of repeated structure: the same brick pattern in thousands of places, the same all-empty subtree in every patch of sky. Structural deduplication (SVDAG and its extensions) exploits this at the tree level, merging duplicates into a shared node. This is covered in depth in [sparse voxel DAGs](../storing/sparse-voxel-dags.md); this page treats it as part of the broader toolbox.
- **Precision nobody will notice.** Attributes like density, color, and distance fields are stored at higher precision than the eye or the algorithm requires. Quantization and block texture formats trade a small, controlled amount of precision for large space savings.

A reader can stop here and hold the right mental model: three kinds of redundancy, three categories of tool, one central tradeoff (smaller = slower to decode).

---

## the lossless toolbox

### runs of identical cells — run-length encoding

In a typical terrain chunk, cells tend to clump: a thick layer of stone, a large pocket of air, a deep water column. Instead of storing every cell individually, you record what the run is and how long it lasts — one (value, count) pair per run of identical values. That is the entire mechanism. The standard term for this is <em>run-length encoding</em>, or RLE.

```
# a column of 32 cells: 12 stone, then 15 air, then 5 water
# uncompressed: 32 entries
# RLE: 3 pairs
[(stone, 12), (air, 15), (water, 5)]
```

The gains on real terrain data are dramatic when runs are long. Compressing in the vertical direction (which aligns with geological layers) can achieve around 65× compression; adding an entropy pass with LZ4 or zstd on top of the RLE output can push this to 600–1200× for highly uniform terrain. Minecraft uses RLE combined with zlib for its on-disk chunk format [1].

The cost is random access. To find the value at a specific cell, you must scan the run list from the beginning until your offset is within a run. That is O(r) in the number of runs, not the O(1) of a flat array. For a chunk full of long runs this is fast in practice; for mixed-content chunks it is slow.

RLE also has a failure mode: alternating content (stone, air, stone, air...) produces a run of length 1 for every cell, and the count adds overhead rather than saving it. The worst case can actually *expand* storage.

#### when to reach for RLE

- Disk serialization and network transmission — it is Minecraft's actual on-disk format, and sequential reads at load time tolerate the lack of random access.
- Sweeping passes over mostly-uniform data (counting occupied cells, computing surface area) where you read every cell in sequence anyway.
- Chunks that are nearly uniform — all air above the world, all stone below — where runs are very long.

#### when RLE loses

- Any workload requiring many random reads: physics, raycasting, neighbor lookups during mesh generation. Each random access requires a list scan.
- High-entropy chunks: dense player builds, mixed geology, generated structures. Short runs mean minimal savings, and the count bytes add overhead.
- As an in-memory working format — keep it for serialization and decode to a flat array or palette store for runtime.

### a small dictionary per region — palette indexing

Most chunks of game terrain use a small number of distinct block types — a handful of stone variants, air, maybe water. Even a complex player build typically uses fewer than 64 distinct materials per 16³ section. The full registry might contain thousands of types, but any one section uses a tiny subset.

Instead of storing the full type ID per cell, you store two things: a small lookup table called the <em>palette</em> containing just the types that actually appear in this chunk, and a tightly bit-packed array of short indices into that palette, one index per cell. The index needs only enough bits to address the palette — with 4 types, 2 bits suffice; with 16 types, 4 bits.

```
# a 16³ section with 5 distinct block types
palette = [air, stone, iron_ore, granite, water]  # 5 entries

# each cell stores a 3-bit index (log₂ of next power-of-two ≥ 5)
# 4096 cells × 3 bits = 12,288 bits = 1,536 bytes
# vs 4096 × 16-bit global IDs = 8,192 bytes → 5× smaller
```

The key property that RLE lacks: random access stays O(1). Looking up cell (x, y, z) is the same index arithmetic as a flat array, followed by one palette lookup. Writes handle palette growth: if a new type is added and the palette overflows its current bit-width, indices are repacked at the next width.

Minecraft transitions from an indirect (local) palette to direct (global) IDs when a section would need more than 8 bits per index — at that threshold, using global IDs directly is no worse than maintaining a local palette [1].

The single-entry optimization is worth noting: when a section contains exactly one block type (an all-air section, a solid-stone layer), the palette has one entry and the index array can be omitted entirely. The section is fully described by one value.

#### direct vs indirect palette

- **Indirect (local) palette** — indices point into a section-local list. Efficient when the section uses few distinct types; typical terrain needs only 2–4 bits per cell.
- **Direct palette** — indices point into the global block registry. Used when the section has too many distinct types for an indirect palette to save space. Random access stays O(1) but bit width grows toward the global ID width.

#### when palette indexing wins

- Game-world terrain: most sections use 4–16 distinct types, comfortably fitting in 4 bits.
- Any workload requiring fast random access (physics, raycasting, neighbor lookup) — unlike RLE.
- Sections with large uniform sub-regions: the single-entry optimization makes all-air and all-stone sections nearly free.

### entropy coding on top

Entropy coding — <em>DEFLATE</em>, <em>LZ4</em>, <em>zstd</em> — is not a replacement for RLE or palette compression; it is a second pass applied to the already-compressed output before writing to disk or sending over a network.

The distinction matters. LZ4 is pure dictionary matching (LZ77) with no entropy coding stage — it decompresses at multiple GB/s per core but does not achieve the highest ratios. DEFLATE adds Huffman coding on top of LZ77 (used in zlib and gzip), achieving smaller output at the cost of slower decompression. Zstd combines LZ77 with finite-state entropy, achieving ratios comparable to DEFLATE at speeds closer to LZ4 — usually the best balance for voxel data [2].

None of these formats support random access within the compressed stream. The entire block must be decompressed to read any cell. This makes them appropriate for cold data (disk, network) but not for in-memory working storage. The correct pattern is: store RLE or palette data in memory for runtime use, apply zstd or LZ4 on top when serializing to disk or wire.

---

## structural compression — deduplicating identical regions

The two techniques above compress data within a chunk. Structural compression works at a larger scale: it finds regions of the volume that are bit-for-bit identical and stores them only once, with multiple parents pointing at the shared copy.

This requires a tree representation. [Sparse voxel DAGs](../storing/sparse-voxel-dags.md) covers the mechanism in full — the summary here is enough to place it in the toolbox.

### merging identical subtrees — the sparse voxel DAG

An SVO (sparse voxel octree) prunes empty space but does nothing about repeated structure. A building with thousands of identical brick subtrees stores each one separately. The insight behind the <em>sparse voxel DAG (SVDAG)</em> is to build the SVO and then deduplicate it bottom-up: hash every node by its children; if two nodes are identical, keep one and point both parents at it. The tree becomes a directed acyclic graph — still no cycles, but now one node can have many parents.

The gains in scenes with real architectural or natural repetition are large. For the EpicCitadel scene voxelized to 128K³ resolution (~19 billion voxels): SVO requires ~5.1 GB on the GPU; SVDAG reduces this to ~945 MB [3]. In the same structural family but at 256K³ resolution, the PowerPlant model with ~100 billion voxels fits in ~575 MB at 0.048 bits per voxel [4].

The crucial property for performance: traversal is identical to an SVO. At each node the ray reads the child mask and follows the pointer — the fact that the pointer is shared with other nodes is invisible to the traversal algorithm. There is no decompression step and no decode cost.

### symmetry-aware merging — SSVDAG

The plain SVDAG only merges subtrees that are bit-for-bit identical. But a left wall and its mirror image are nearly identical — they differ only in which axis the child ordering is flipped along. The <em>symmetry-aware sparse voxel DAG (SSVDAG)</em> extends merging to subtrees that are identical under axis-aligned plane reflections [4].

Three bits are added to each child pointer — one per axis — indicating that the child should be read with that axis mirrored. A subtree that would have been stored separately as the mirror of another node now shares one underlying node with a mirroring flag set.

For the EpicCitadel scene, SVDAG uses ~167 MB; SSVDAG reduces this to ~86 MB at 0.123 bits per voxel. The PowerPlant result is 0.048 bits per voxel. This is roughly an 11× reduction over a plain SVDAG on architectural geometry [4].

### attribute compression on the DAG — colors

The geometry DAG merges nodes that have the same occupancy pattern. Color breaks this: a red brick and a blue brick share a shape but not a color, so they cannot share a node. Naively storing color in the DAG undoes most of the structural compression.

The approach from Dolonius, Sintorn, Kämpe, and Assarsson decouples geometry from color entirely [5]:

- The geometry DAG is built and compressed normally — no change to structural sharing.
- Each leaf in the DAG receives a pointer into a separate 1D array of colors, one entry per surface voxel.
- That 1D array is reshaped into a 2D image using a space-filling curve (which places nearby voxels near each other in the image, preserving locality).
- The 2D image is compressed with standard GPU texture compression — BC7 or ASTC — which the GPU decodes for free in a shader.

The geometry graph stays fully shared; the color layer pays only for surface voxels with distinct colors. Combined with BC7/ASTC, this achieves roughly 3× additional compression on the color data with very little perceptual loss [5]. The same principle applies to any per-voxel attribute that varies independently of occupancy: normals, material IDs, emissivity all require a similar decoupled layer.

---

## attribute compression — precision beyond what's needed

### quantizing density and SDF values

A signed distance field or a smoke density volume is typically stored as 32-bit floats. Most applications don't need that precision. A renderer that samples a density volume at most samples 256 levels of opacity per ray step; a physics simulation that uses an SDF for collision detection only needs centimeter-level accuracy at meter-scale objects.

Reducing from 32-bit float to 16-bit float (half precision) halves the memory cost with minimal quality impact for smooth volumes. Going further to 8-bit fixed-point cuts memory by 4× — useful for density fields and occupancy values where the range is bounded and the precision requirement is low.

NanoVDB, the GPU-resident form of the VDB format, supports blocked floating-point quantization at 2, 4, 8, and 16 bits per voxel. On real volumetric production data, this typically reduces memory footprints by 4–6× relative to uncompressed VDB, and because the resulting data is more cache-friendly, rendering can actually run 10–30% faster despite the per-voxel decode cost — the workload is memory-bound, not compute-bound [6].

### block texture formats for color data

The GPU natively decompresses <em>block compression</em> formats — fixed-size compressed blocks of texels decoded in hardware with no visible shader cost. When voxel color data is stored as a 2D texture (as in the colored-DAG approach), these formats apply directly.

The main formats:

- **BC4 / BC5** — single or dual channel. 4 bits per texel. Hardware everywhere. For luminance-only or two-channel data (e.g. compressed normal maps).
- **BC7** — up to four channels, 8 bits per texel fixed (128 bits per 4×4 block). High quality RGBA. Supported on all modern desktop GPUs.
- **ASTC** — variable block size (4×4 to 12×12 texels), 128 bits per block, so bit rates range from 8 bpt down to 0.89 bpt. Supports 3D textures natively. Standard on mobile GPUs; supported on modern desktop via extensions.

At 8 bpt, BC7 and ASTC 4×4 offer comparable quality. ASTC's variable bit rate and 3D texture support make it particularly relevant for volumetric voxel data. Both decode in hardware — zero shader instructions, zero bandwidth to the shader for the compressed bytes themselves.

---

## the central tension — size vs. access cost

Every compression choice you make sits on a spectrum between two extremes: a flat uncompressed array you can read in one memory access, and a maximally compressed representation that requires significant work to access any single cell. The right point on that spectrum depends on three questions:

1. **Is the bottleneck memory or compute?** If the GPU is memory-bandwidth-limited (reading more data than the bus can carry), shrinking the data with even a costly decode wins. If the GPU is compute-limited, adding decode cost to an already-stressed pipeline hurts.
2. **Is access random or sequential?** A ray-marcher samples arbitrary positions across the volume — random access. A mesh generator sweeps every cell in a chunk in order — sequential. RLE and entropy coding are efficient for sequential workloads and terrible for random ones.
3. **Does the data need to be written?** SVDAGs achieve their compression by sharing subtrees, and editing one voxel may require splitting a shared node and cascading changes up the tree. In practice, SVDAGs are read-only stores used for rendering; editable worlds need a more mutable structure.

| technique | compression ratio | random-access cost | editable? | best fit |
|---|---|---|---|---|
| flat array (no compression) | 1× | O(1), one lookup | yes | physics, editable world |
| palette + indexed | 2–10× typical | O(1) + one palette lookup | yes (with repacking) | in-memory game chunks |
| RLE | 10–600× depending on uniformity | O(r) — scan runs | slow (splits runs on write) | disk serialization, uniform data |
| RLE + zstd/LZ4 | 100–1200× for uniform terrain | none — full decode required | no | cold storage, network transfer |
| SVDAG (structural) | ~5× over SVO, 50–100× over dense | O(log n), no decompression | no (without HashDAG) | static scene, ray-traced rendering |
| SSVDAG | ~11× over SVDAG | O(log n), no decompression | no | high-rep architectural scenes |
| quantized attributes | 4–6× for float data | O(1) + unpack (cheap) | yes | density fields, SDF, colors |
| BC7/ASTC | ~6–8× for RGBA textures | O(1), hardware decode | no | color layer on DAG, texture atlases |

The performance budget perspective — which workloads are memory-bound vs. compute-bound on real hardware — is covered in [the performance budget](./the-performance-budget.md). Memory layout choices that affect which cells you pay cache-miss penalties to access are in [memory layout and Morton order](./memory-layout-and-morton.md).

---

## lossy vs. lossless — when each is acceptable

Every technique above except quantization and BC7/ASTC is lossless: the decompressed data is bit-for-bit identical to the input. Palette compression, RLE, and structural DAG deduplication all preserve the original voxel values exactly.

Quantization and block texture formats are <em>lossy</em>: the decompressed value is close to the original, but not identical. Whether that difference is acceptable depends entirely on what the data is used for.

**Game worlds and visual content:** Lossy is usually fine. A voxel color compressed with BC7 at 8 bpt introduces tiny errors that are indistinguishable from noise at typical viewing distances. A density field quantized to 16-bit half-float renders smoke that looks identical to 32-bit. The world doesn't need to be pixel-perfect — it needs to look right and play right. Even geometry can be lossy: later work after Kämpe et al. showed that modifying 1–5% of voxels produces ~10–50% further compression on top of the SVDAG with often imperceptible visual impact [7].

**Medical imaging (CT, MRI, PET):** Lossless or near-lossless is the default requirement. A density value that is subtly wrong can mean a missed lesion or a measurement error in a treatment plan. The standard approach is lossless compression (ratio 2–4×) or near-lossless quantization with a mathematically bounded maximum per-voxel error. A hybrid model is sometimes used: lossless compression for the region of interest (the tumor, the organ being studied), lossy compression for surrounding tissue where diagnostic precision is lower-priority [8].

**Industrial and CAD models:** Geometry must be exact. A surface that is off by even a fraction of a millimeter can invalidate a fit check or a structural simulation. Geometry stays lossless; color or metadata may be quantized if it does not affect the verification pipeline.

The rule is: lossy compression is acceptable when the downstream use of the data tolerates error in that attribute at the spatial scale of the error. When in doubt, keep it lossless and accept the lower compression ratio.

---

## how the techniques connect

These techniques are not mutually exclusive — they stack. A real system uses multiple layers:

1. A chunked dense grid ([dense grids and chunks](../storing/dense-grids-and-chunks.md)) provides the in-memory working store for editable terrain. Each chunk uses palette indexing for fast random access.
2. When a chunk is serialized to disk, RLE runs are identified and zstd is applied on top — the cold storage representation is far smaller.
3. For a static read-only scene (a building, a landscape, a large prop), the geometry is baked into an SVDAG or SSVDAG for GPU rendering without needing the editable layer at all.
4. Color and other attributes, whether on a chunk or a DAG, are stored in a decoupled texture and BC7/ASTC-compressed for hardware decode.
5. Density or SDF attributes that don't need float precision are quantized at the appropriate bit depth.

How streaming fits in — when chunks are loaded into VRAM, and how an LOD system determines which resolution is needed — is in [chunk management and streaming](../engines/chunk-management-and-streaming.md).

---

## references

[1] Mojang. (2024). "Chunk Format." *Minecraft Wiki*. https://minecraft.wiki/w/Chunk_format — canonical documentation of the indirect/direct palette scheme, bit-depth calculation, and RLE+zlib on-disk format.

[2] Facebook Open Source. (2024). "Zstandard — Real-time lossless compression algorithm." *GitHub*. https://github.com/facebook/zstd — Zstd algorithm combining LZ77 with FSE, offering zlib-comparable ratios at near-LZ4 speeds.

[3] Kämpe, V., Sintorn, E., and Assarsson, U. (2013). "High Resolution Sparse Voxel DAGs." *ACM Transactions on Graphics*, 32(4), Article 124. DOI: 10.1145/2461912.2462024. [local PDF](../papers/kampe-sintorn-assarsson-2013-high-resolution-sparse-voxel-dags.pdf) · [source](https://www.cse.chalmers.se/~uffe/HighResolutionSparseVoxelDAGs.pdf)

[4] Villanueva, A. J., Marton, F., and Gobbetti, E. (2016). "SSVDAGs: symmetry-aware sparse voxel DAGs." *Proceedings of the 20th ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (i3D 2016)*, pp. 7–14. DOI: 10.1145/2856400.2856420. [local PDF](../papers/villanueva-marton-gobbetti-2016-ssvdag-symmetry-aware-sparse-voxel-dags.pdf) · [source](https://www.crs4.it/vic/data/papers/i3d2016-symmetry-dags.pdf)

[5] Dolonius, D., Sintorn, E., Kämpe, V., and Assarsson, U. (2017). "Compressing color data for voxelized surface geometry." *Proceedings of the 21st ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (i3D 2017)* (Best Paper). Extended version in *IEEE Transactions on Visualization and Computer Graphics*, 25(2), pp. 1270–1282 (2019). DOI: 10.1109/TVCG.2017.2741480. [local PDF](../papers/dolonius-sintorn-kampe-assarsson-2017-compressing-color-voxelized-surface-geometry.pdf) · [source](https://www.cse.chalmers.se/~uffe/dolonius2017i3d.pdf)

[6] Museth, K. (2021). "NanoVDB: A GPU-Friendly and Portable VDB Data Structure for Real-Time Rendering and Simulation." *ACM SIGGRAPH 2021 Talks*, Article 1. DOI: 10.1145/3450623.3464653. [local PDF](../papers/museth-2021-nanovdb-gpu-friendly-portable-vdb.pdf) · [source](https://dl.acm.org/doi/fullHtml/10.1145/3450623.3464653)

[7] Scandolo, L., Bauszat, P., and Eisemann, E. (2020). "Lossy Geometry Compression for High Resolution Voxel Scenes." *Proceedings of the ACM on Computer Graphics and Interactive Techniques*, 3(1), Article 7. DOI: 10.1145/3384541. (Demonstrates 10–50% additional compression on top of SVDAG by modifying 1–5% of voxels.) [source](https://dl.acm.org/doi/10.1145/3384541)

[8] Subramanian, N. and Umamaheshwari, A. (2021). "On a hybrid lossless compression technique for three-dimensional medical images." *Journal of Applied Clinical Medical Physics*, 22(4), pp. 270–278. DOI: 10.1002/acm2.12960. (Near-lossless and hybrid ROI-lossless approaches for CT/MRI volumetric data.) [source](https://aapm.onlinelibrary.wiley.com/doi/full/10.1002/acm2.12960)
