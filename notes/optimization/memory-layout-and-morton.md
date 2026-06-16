<link rel="stylesheet" href="./css/globals.css">

# memory layout and Morton order

Imagine two voxel engines running the same raymarching loop over the same 256³ grid. One runs in 8 ms. The other runs in 48 ms. The algorithm is identical. The only difference is the order in which voxels sit in memory.

That is not a thought experiment — it describes the real difference between a naively laid-out flat array and a Morton-ordered one on a memory-bound workload. Voxel work is almost always memory-bound: the bottleneck is how fast you can feed data from RAM to the processor, not how fast the processor does arithmetic. The [performance budget](./the-performance-budget.md) for any voxel system has memory bandwidth as its single biggest line item. Layout is how you spend that budget wisely.

---

## the core idea — bring memory and neighborhood into alignment

When your code visits a voxel, the CPU or GPU does not fetch that one value from RAM. It fetches a contiguous block of 64 bytes — a <em>cache line</em> — containing the target voxel plus its neighbors in memory. If those memory-neighbors happen to be the same cells you visit next, the hardware loads them for free, already warm in the cache. If they are unrelated cells you will never touch again, the load is wasted and the next voxel requires another trip to RAM.

The word for how well "neighbors in memory" matches "neighbors in space" is <em>spatial locality</em>. Good locality means most of the cache line you load gets used before it is evicted. Poor locality means you pay for a 64-byte line and use 4 bytes of it.

---

## why the standard layout has poor locality

The natural way to store a 3D grid is a flat array indexed by a formula like:

```
index = x * (Y * Z) + y * Z + z
```

This is <em>row-major order</em>: Z changes fastest, then Y, then X. That means voxels at `(x, y, z)` and `(x, y, z+1)` are adjacent in memory. That's the only neighbor pair that is adjacent. The six face-neighbors of any voxel at `(x, y, z)` are:

| neighbor | memory distance |
|----------|----------------|
| `(x, y, z+1)` | 1 element — same cache line |
| `(x, y, z-1)` | 1 element — same cache line |
| `(x, y+1, z)` | `Z` elements apart |
| `(x, y-1, z)` | `Z` elements apart |
| `(x+1, y, z)` | `Y*Z` elements apart |
| `(x-1, y, z)` | `Y*Z` elements apart |

For a 256³ grid, the X-neighbor is 65 536 elements away. No cache line spans that distance. Every time you touch an X- or Y-neighbor, you load a fresh cache line and use one value from it. Algorithms that touch all six neighbors — finite-difference simulations, ambient occlusion, smoothing — pay the cache-miss penalty five times per voxel per step.

---

## the fix — interleave bits to bring 3D neighbors close in 1D

The root problem is that row-major order has only one "fast axis." Interleaving the bits of the three coordinates solves this by distributing locality across all three axes.

Take the x, y, and z coordinates and, instead of combining them arithmetically, alternate their bits in the output index. For a 3D point, the bit pattern of the Morton index looks like:

```
Morton index bits: ... x₂ y₂ z₂ x₁ y₁ z₁ x₀ y₀ z₀
```

where `x₀` is the lowest bit of x, `y₀` of y, and so on. Every group of three consecutive bits in the Morton index encodes one step along each axis simultaneously. Because of this, two cells that differ by 1 in any single coordinate are at most a small Morton-distance apart. Cells that are near each other in 3D space land near each other in the 1D array — they often share a cache line or sit within a few lines of each other.

This traversal pattern — threading through 3D space in a shape that recursively subdivides each octant before moving to the next — is a <em>Z-order curve</em>, also called a <em>Morton curve</em> or <em>Morton order</em>, after G. M. Morton who introduced the idea for geographic file sequencing in a 1966 IBM technical report [1]. The name "Z-order" comes from the Z-shape the path traces when drawn across a 2×2 block.

#### computing a Morton index

There are three practical approaches, trading readability for speed:

**Loop method** — iterate over each bit position, extract bits from x, y, and z, and place them in the result. Simple to understand; roughly 12× slower than the alternatives for a 256³ grid.

**Magic-bits method** — use a sequence of shift-and-mask operations with fixed bit patterns (the "B array" constants 0x55555555, 0x33333333, and so on from Stanford's Bit Twiddling Hacks [2]) to spread each coordinate's bits into every third position in one pass. Each stage doubles the spacing between bits until they land in their final slots. About 12× faster than the loop [3].

```c
// spread the bits of x into every 3rd position (magic-bits, 21-bit input)
uint32_t split_by_3(uint32_t x) {
    x &= 0x1fffff;
    x = (x | (x << 32)) & 0x1f00000000ffff;
    x = (x | (x << 16)) & 0x1f0000ff0000ff;
    x = (x | (x << 8))  & 0x100f00f00f00f00f;
    x = (x | (x << 4))  & 0x10c30c30c30c30c3;
    x = (x | (x << 2))  & 0x1249249249249249;
    return x;
}
uint64_t morton3D(uint32_t x, uint32_t y, uint32_t z) {
    return split_by_3(x) | (split_by_3(y) << 1) | (split_by_3(z) << 2);
}
```

**Lookup table (LUT) method** — precompute Morton indices for all 256 possible byte values, then assemble the full index by combining table lookups with bit shifts. Fastest traditional method; around 4× faster than the magic-bits approach [3].

**BMI2 `PDEP` instruction** — on modern x86 CPUs, a single hardware instruction deposits bits into a pattern, doing the entire spread in one operation. Roughly 3× faster than the LUT method where available [3].

The crossover point matters: the overhead of computing a 3D Morton index is now less than the cost of a single cache miss on most hardware. The index computation is always worth it when it prevents even one cache miss per voxel.

---

## two coarser approaches to the same goal

Morton order works at the individual-voxel level. Two broader techniques achieve the same locality improvement at the block level.

### tiling and bricking

Instead of reordering individual voxels, divide the grid into fixed-size rectangular bricks — commonly 8×8×8 or 16×16×16 — and store each brick contiguously in memory. Within a brick, voxels use ordinary row-major order. Bricks are themselves arranged in a flat outer array (or a Morton-ordered one for nested locality).

The result is that all 512 voxels of an 8³ brick fit in 32 consecutive 64-byte cache lines (for single-byte occupancy) or 128 lines (for 4-byte floats). Any algorithm whose working set stays within one or a few bricks — local smoothing, stencil evaluation, AO sampling — touches only warm cache lines. This is exactly the organization OpenVDB uses: its leaf nodes are fixed 8×8×8 voxel blocks, each stored contiguously, so the framework can "efficiently utilize typical cache architectures" for spatially coherent access patterns [4]. See [hash grids and bricks](../storing/hash-grids-and-bricks.md) and [dense grids and chunks](../storing/dense-grids-and-chunks.md) for how this plays out in practice.

### Hilbert curves

The Hilbert curve is a space-filling curve with strictly better locality than Z-order: it never makes diagonal jumps between octants. In Z-order, roughly every fourth step crosses a larger region boundary, creating a short burst of distant memory accesses. The Hilbert curve eliminates those jumps entirely [5].

The tradeoff is computation cost: computing a Hilbert index requires a recursive digit-transformation algorithm that cannot be reduced to simple bit interleaving. In practice, the Hilbert curve is used in offline pipelines (data compression, point-cloud preprocessing) where its superior locality justifies the extra work, while Z-order dominates real-time voxel systems where index computation happens per frame.

| curve | locality quality | index cost | hardware optimization |
|-------|-----------------|------------|-----------------------|
| Z-order / Morton | good | low (bit interleaving) | BMI2 PDEP, trivial |
| Hilbert | excellent | high (recursive) | none available |
| row-major | poor (one axis only) | trivial | — |

---

## payload layout — AoS vs SoA

Morton order determines *where* each voxel lives. The second decision is *how* the data inside each voxel is organized.

Suppose each voxel carries four fields: a density value, a material ID, a temperature, and an occupancy flag. There are two ways to lay this out for a grid of N voxels.

**Array of Structs (AoS)** keeps all four fields together per voxel:

```
[density₀, material₀, temp₀, occ₀ | density₁, material₁, temp₁, occ₁ | ...]
```

**Struct of Arrays (SoA)** groups all values of each field together:

```
[density₀, density₁, density₂, ... | material₀, material₁, ... | ...]
```

The choice determines what ends up on a cache line — and therefore what an algorithm can process cheaply.

<em>AoS</em> loads well when you need all fields of one voxel at a time: rendering a single voxel that needs density, material, and color reads one cache line and gets everything. It also maps naturally to C structs.

<em>SoA</em> loads well when you need one field across many voxels: a simulation pass that updates temperature for a region of 64 voxels loads 64 consecutive temperature values from one array, filling entire cache lines with useful data and nothing else. This pattern is essential for SIMD processing — the CPU can load eight floats at once from a float array and apply one operation to all of them. On a GPU, where a warp of 32 threads executes in lockstep, SoA means thread 0 reads `density[0]`, thread 1 reads `density[1]`, and so on — 32 consecutive addresses combine into a single coalesced memory transaction. With AoS, those same 32 reads are strided (each 16 or 32 bytes apart), forcing 32 separate transactions. The bandwidth difference can reach 5–20× [6].

#### when to reach for each

- **AoS** when the dominant access pattern touches all fields per voxel; cache is large enough to amortize strided reads; code maintainability matters more than SIMD throughput.
- **SoA** when a pass operates on one or two fields at a time across many voxels; GPU compute is involved; SIMD vectorization is the target.
- **AoSoA** (Array of Structs of Arrays) when you need both: tile the data in groups matching the SIMD width (8 floats for AVX, 4 for SSE), keep fields grouped within each tile. This hybrid preserves SoA's coalescing while being more cache-friendly for the outer loop. It is common in game physics engines and GPU fluid solvers.

The SoA advantage is most pronounced in GPU voxel work — see [gpu voxel techniques](./gpu-voxel-techniques.md) for how coalesced access patterns interact with shader design. Compression schemes that operate field-by-field also strongly prefer SoA; the [compression techniques](./compression-techniques.md) page covers that connection.

---

## putting it together — layout is a design decision

The [voxel grid](../foundations/the-voxel-grid.md) page establishes that a voxel does not store its own position — its location is implicit in its index. That economy of representation is exactly what makes layout decisions so powerful: by controlling what index a voxel gets and how its fields are arranged in memory, you control what the hardware sees in its cache hierarchy without changing any algorithm logic.

The decisions compound:

1. Use Morton order (or bricking) so that spatially nearby voxels are nearby in memory — this is where the 6× throughput gains live for memory-bound workloads.
2. Choose SoA over AoS when GPU compute or SIMD is in the path — this is where coalescing gains live.
3. Let the structure (VDB leaf nodes, hash-grid bricks) enforce the layout automatically so application code doesn't have to manage it — covered in [hash grids and bricks](../storing/hash-grids-and-bricks.md).

A voxel system that gets all three right runs the same algorithm at memory-bandwidth efficiency rather than at cache-miss penalty speed. That is what the 6× difference in the opening example is made of.

---

## references

[1] Morton, G. M. (1966). "A Computer Oriented Geodetic Data Base and a New Technique in File Sequencing." IBM Ltd., Ottawa, Canada. (Technical report. Paywalled / internal IBM document. Canonical citation for Z-order curves; PDF circulates via IBM research archive at `domino.research.ibm.com`.)

[2] Anderson, S. E. (2005). "Bit Twiddling Hacks." Stanford University Graphics Group. [source](https://graphics.stanford.edu/~seander/bithacks.html) — open-access reference for the magic-bits / binary-magic-number interleaving method.

[3] Baert, J. (2013). "Morton Encoding/Decoding Through Bit Interleaving: Implementations." Forceflow.be blog, October 7, 2013. [source](https://www.forceflow.be/2013/10/07/morton-encodingdecoding-through-bit-interleaving-implementations/) — benchmarks comparing loop, magic-bits, LUT, and BMI2 methods for 256³ grids.

[4] Museth, K. (2013). "VDB: High-Resolution Sparse Volumes with Dynamic Topology." *ACM Transactions on Graphics*, 32(3), Article 27. DOI: 10.1145/2487228.2487235. [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf) · [source](https://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf) — the foundational VDB paper; Section 3 describes the fixed 8×8×8 leaf node layout and cache-coherent accessor design.

[5] Eisenwave (2024). "Space-Filling Curves." Voxel Compression Docs. [source](https://eisenwave.github.io/voxel-compression-docs/rle/space_filling_curves.html) — empirical comparison of nested iteration, Z-order, and Hilbert traversal on voxel data; Hilbert eliminates diagonal jumps at higher implementation cost.

[6] Wald, I., et al. (2014). "SIMD Ray Stream Tracing — SIMD Ray Traversal with Generalized Ray Packets." *Proceedings of High-Performance Graphics*. — AoS vs SoA layout impact; 5–20× coalescing difference on GPU memory transactions. (See also: Wikipedia, "AoS and SoA", [source](https://en.wikipedia.org/wiki/AoS_and_SoA), for an accessible survey of the layout tradeoffs.)

[7] Choi, S., Park, D.-G., Hwang, S.-Y., and Kim, T.-W. (2025). "Surfel-LIO: Fast LiDAR-Inertial Odometry with Pre-computed Surfels and Hierarchical Z-order Voxel Hashing." *arXiv:2512.03397* [cs.RO]. CC BY 4.0. [source](https://arxiv.org/abs/2512.03397) — practical application of Z-order Morton hashing for cache-friendly voxel spatial indexing; notes that cache hits cost ~4 cycles vs ~100 cycles for cache misses in random access patterns.
