<link rel="stylesheet" href="./css/globals.css">

# the storage problem

You want to build a large open world, or simulate a high-resolution fluid volume, or reconstruct a room from a depth sensor in real time. The obvious approach — a flat 3D array with one slot per cell — fails quickly. At moderate resolution it asks for more RAM or VRAM than the hardware has. At high resolution it isn't even close.

Every storage structure covered in this domain is a response to that failure. Understanding *why* the flat array fails, and which properties of real volumes let you escape it, is the mental model that makes every subsequent choice legible.

---

## the coarse model

A [voxel grid](../foundations/the-voxel-grid.md) is a 3D array. If you lay it out flat — one contiguous slot for every possible cell — you pay for *every cell*, occupied or not. The number of cells is the product of the three axis counts, so it grows as the cube of linear resolution. Double the resolution in each axis and you need eight times the memory.

Two facts about real volumes let you escape this:

- Most cells in a scene are empty or uniform. The surface of a rock, a cloud of smoke, a built environment — these are thin shells in a mostly-empty volume. The cells that carry information are a small minority. This is <em>sparsity</em>.
- Detail you can see clearly matters; detail far away or behind walls does not. Representing distant regions at coarser resolution loses nothing perceptible while cutting memory steeply. This is <em>level of detail</em>, or LOD — the same word, from different angles, is also called hierarchy.

Every voxel storage structure in this domain exploits one or both of those facts. The rest of this page makes the failure case concrete and maps out how each lever helps.

---

## why the flat array fails — the cube law, made concrete

The number of cells in a uniform grid of side length *n* is *n × n × n = n³*. Every time you double *n* you multiply the cell count by 2³ = 8.

That multiplier is relentless. Here are real numbers, assuming common payload sizes drawn from the data models in [voxel data models](../foundations/voxel-data-models.md):

| resolution | cells | 1 byte (occupancy / material ID) | 4 bytes (float density or RGBA) | 32 bytes (SDF + normal + material) |
|---|---|---|---|---|
| 256³ | ~16.8 M | **16 MB** | **64 MB** | **512 MB** |
| 512³ | ~134 M | **128 MB** | **512 MB** | **4 GB** |
| 1024³ | ~1.07 B | **1 GB** | **4 GB** | **32 GB** |

A single 512³ grid of 4-byte floats already fills 512 MB — a large chunk of a GPU's VRAM budget, before any geometry, textures, or render targets. A 1024³ grid of even the most minimal payload exceeds 1 GB. Attach per-voxel normals, color, and material data and it reaches tens of gigabytes at that resolution.

The cube law has two faces worth naming separately.

### doubling resolution multiplies cost by 8

Going from 256³ to 512³ in all three axes gives you 8× more cells per unit of linear detail. This is why "just raise the resolution" is not a free option. An artist who wants to see features at half the current scale asks, perhaps without realizing it, for eight times the storage.

### the world is mostly empty

A dense grid pays for air, rock interior, and deep ocean floor with the same per-cell cost as the surface where everything interesting happens. In typical 3D scenes — outdoor environments, constructed spaces, mechanical assemblies — over 90% of cells are empty or uniform [1]. The flat array charges you for all of them.

---

## the two facts that rescue you

### sparsity — only store what's there

If the occupied cells are a small fraction of the total, you only need a structure that allocates memory for cells that carry information and ignores the rest. A structure that does this is called <em>sparse</em> — it skips the zeros rather than storing them explicitly.

The catch: you can no longer look up a cell by computing `array[x][y][z]` in one step. You need an index, a tree traversal, or a hash lookup to find whether a cell exists and where its data lives. Sparsity trades memory for access cost.

Real volumes can be very sparse. Museth's VDB structure, used in visual effects production, was designed specifically around the observation that volumetric data in practice is overwhelmingly empty — the paper demonstrates that the hierarchical B-tree layout captures the occupied fraction at a fraction of the dense cost [2]. The [OpenVDB and NanoVDB](./openvdb-and-nanovdb.md) page covers how that structure works.

### hierarchy and LOD — only store detail where it's needed

Even the occupied cells don't all need the same resolution. A mountain range 5 km away can be represented at a cell size 100× larger than the rocks at your feet; the perceptual difference is invisible. A hierarchical structure encodes coarser representations for distant or unvisited regions and finer ones for regions that need them.

A tree structure naturally provides this: the root covers the whole volume at low resolution; each level down doubles the resolution over a smaller region. You spend memory budget where detail is actually needed and save it everywhere else. [Octrees and SVOs](./octrees-and-svo.md) covers the main tree-based approach. LOD and streaming together let a large world fit in a fixed memory budget: only the near region is held at full resolution while the rest is either coarser or not resident at all — see [chunk management and streaming](../engines/chunk-management-and-streaming.md).

---

## the levers the rest of this domain pulls

Each page in the STORING VOXELS domain works one or more of the following knobs. Understanding which knob a structure turns — and at what cost — is the whole lesson.

- **Sparsity.** Only allocate memory for occupied or non-uniform cells. Structures: [octrees / SVOs](./octrees-and-svo.md), [hash grids and bricks](./hash-grids-and-bricks.md), [sparse voxel DAGs](./sparse-voxel-dags.md), [OpenVDB / NanoVDB](./openvdb-and-nanovdb.md).
- **Hierarchy.** Represent coarser detail at higher levels of a tree; refine only where needed. All tree-based structures do this. SVOs and VDB are the canonical examples.
- **Redundancy elimination.** Some regions of a volume look identical. A directed acyclic graph (DAG) instead of a tree merges those identical subtrees into a single shared node in memory. This is what [sparse voxel DAGs](./sparse-voxel-dags.md) add on top of an SVO — Kämpe et al. showed that for typical scenes, converting an SVO to a DAG reduces node count by one to three orders of magnitude [3].
- **Chunking.** Divide the world into fixed-size blocks. Each chunk can be loaded, unloaded, and processed independently, making infinite worlds tractable. [Dense grids and chunks](./dense-grids-and-chunks.md) is the starting point; it shows why chunked dense storage is often the right answer for editable voxel games.
- **Compression.** Within a chunk or leaf node, runs of identical values can be stored compactly. [Compression techniques](../optimization/compression-techniques.md) covers the options: RLE, bit-packing, and more aggressive schemes.
- **GPU residency and streaming.** Not everything fits in VRAM at once. Only the data the camera needs this frame needs to be on the GPU. The rest lives in system RAM or on disk and is streamed in as needed. [Chunk management and streaming](../engines/chunk-management-and-streaming.md) is where this mechanism lives.

---

## the honest tradeoff every structure makes

There is no free lunch. Every structure picks a point in a three-way tradeoff:

| | memory | random access speed | edit speed |
|---|---|---|---|
| dense flat array | worst (O(n³)) | best (O(1), one multiply) | best (O(1), write one slot) |
| chunked dense grid | moderate (pays for full chunks) | very fast (O(1) within chunk) | very fast |
| spatial hash | low (only occupied bricks) | fast average (O(1) amortized) | fast |
| octree / SVO | low (sparse) | moderate (O(log n) tree walk) | slow (subtree restructure) |
| SVO + contour / VDB | low to very low | moderate | moderate |
| DAG (SVDAG) | very low (deduplication) | moderate (O(log n), shared nodes) | very slow (shared nodes break on edit) |

A few points worth spelling out:

**Random access vs. tree traversal.** A dense array reaches any cell in one arithmetic step. A tree requires walking from the root, one pointer per level — typically log₂(n) steps for a resolution of n [4]. For a 1024³ grid that is about 10 steps. That overhead matters enormously when you have millions of ray-march queries per frame.

**Edit speed vs. compression.** A DAG achieves its dramatic compression by sharing identical subtrees. The moment you write to one cell you potentially need to split a shared node, which is expensive and can cascade. SVDAGs are effectively read-only in practice — you render from them but you don't edit them in place [3]. If your application needs interactive edits, you work with a less compressed structure and convert to a DAG for read-only rendering passes.

**Memory vs. GPU friendliness.** Pointer-heavy tree structures are difficult to pack efficiently for the GPU, which favors flat, strided arrays. Brick-based structures (see [hash grids and bricks](./hash-grids-and-bricks.md)) are a pragmatic middle ground: a sparse index in CPU memory points to dense leaf bricks in GPU memory, giving the GPU contiguous data to work with while still skipping empty space at a coarser level.

Niessner et al.'s voxel hashing approach demonstrated this concretely: by hashing only the occupied surface blocks into a flat GPU buffer, the system fused depth camera streams in real time without any hierarchical structure at all — the hash table provided near O(1) access and the entire occupied surface fit in a fraction of the memory a dense grid would need [5].

**The choice is the lesson.** The right structure depends on your access pattern, your edit rate, and whether the bottleneck is CPU memory, VRAM, or memory bandwidth. [Choosing a voxel store](./choosing-a-voxel-store.md) walks through that decision in full, once you know what each structure actually costs.

---

## references

[1] Sparsity-Aware Voxel Attention and Foreground Modulation for 3D Semantic Scene Completion (2025). arXiv:2604.05780. Reports that over 93% of voxels in typical 3D scene understanding benchmarks are empty, driving adoption of sparse representations. [source](https://arxiv.org/abs/2604.05780)

[2] Museth, K. (2013). "VDB: High-Resolution Sparse Volumes with Dynamic Topology." *ACM Transactions on Graphics*, 32(3), Article 27. DOI: 10.1145/2487228.2487235. [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf) · [source](https://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf)

[3] Kämpe, V., Sintorn, E., and Assarsson, U. (2013). "High Resolution Sparse Voxel DAGs." *ACM Transactions on Graphics*, 32(4), Article 101. DOI: 10.1145/2461912.2462024. [local PDF](../papers/kampe-sintorn-assarsson-2013-high-resolution-sparse-voxel-dags.pdf) · [source](https://www.cse.chalmers.se/~uffe/HighResolutionSparseVoxelDAGs.pdf)

[4] Laine, S. and Karras, T. (2010). "Efficient Sparse Voxel Octrees." *Proceedings of the ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (I3D 2010)*. DOI: 10.1145/1730804.1730814. [local PDF](../papers/laine-karras-2010-efficient-sparse-voxel-octrees.pdf) · [source](https://research.nvidia.com/sites/default/files/pubs/2010-02_Efficient-Sparse-Voxel/laine2010i3d_paper.pdf)

[5] Nießner, M., Zollhöfer, M., Izadi, S., and Stamminger, M. (2013). "Real-Time 3D Reconstruction at Scale Using Voxel Hashing." *ACM Transactions on Graphics*, 32(6), Article 169. DOI: 10.1145/2508363.2508374. [source](https://dl.acm.org/doi/10.1145/2508363.2508374) (paywalled — no local PDF)
