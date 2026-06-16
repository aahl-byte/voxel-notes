<link rel="stylesheet" href="./css/globals.css">

# choosing a voxel store

You have a project. Maybe it's a block-building game where players carve and place at runtime. Maybe it's a CT volume where a surgeon needs to inspect arbitrary cross-sections. Maybe it's a cloud of smoke that a GPU ray-marcher has to traverse in milliseconds. Each of these has a structure that fits it well and structures that will fight you every step — the job of this page is to help you pick the right one before you've built the wrong thing.

The five structure pages in this domain each go deep on how one approach works. This page is deliberately shallower on mechanics and deeper on decisions: what pulls you toward each structure, what pulls you away, and what combinations real systems actually use.

---

## the decision axes

Before reaching for any structure, answer six questions about your project. The answers will usually point clearly at one or two candidates.

### how empty is your scene?

The [storage problem](./the-storage-problem.md) is fundamentally a problem of paying for empty space. A naive dense grid allocates memory for every cell in a bounding box — even the 99 % that contain nothing but air. The more empty your scene, the harder a dense structure gets punished on memory.

- **Mostly full** (terrain cross-sections, solid-object interiors, small bounded volumes): dense storage is competitive; the overhead of a sparse structure buys you little.
- **Moderately sparse** (open-world terrain with sky above): chunked dense grids win because active chunks stay dense while absent chunks cost nothing.
- **Highly sparse** (a single thin surface in empty space, medical organ embedded in air, smoke dispersed through a large bounding box): sparse trees and VDB pay per occupied node rather than per bounding box cell. The savings can be 100× or more.

### is the data static or does it change at runtime?

This is the axis that most strongly narrows the field.

- **Static** (pre-baked scene, offline render asset, scanned geometry): you can afford expensive one-time compression. <em>SVDAG</em> and NanoVDB shine here — they are deliberately read-only, and that constraint is what enables their compression.
- **Dynamic / user-editable** (block-building game, destructible terrain, fluid simulation): you need writable nodes. Dense/chunked arrays and OpenVDB support per-voxel writes. SVDAG requires partial rebuild; NanoVDB is fully read-only.
- **Procedurally updated but not user-edited** (probe cache for GI, robot map built from sensor streams): mutable but with predictable write patterns — hash grids and SVO/OctoMap handle this well.

### what does your access pattern look like?

How your code reads the data shapes which layout performs.

- **Random access** — "give me the voxel at world position (x, y, z)" — favors flat arrays (O(1) index arithmetic) and hash grids (O(1) hash lookup). Tree traversal adds log-depth overhead.
- **Ray traversal** — shooting a ray through the volume, skipping empty space, hitting the first occupied cell — favors hierarchical structures (SVO, DAG, VDB) that encode emptiness at multiple scales, enabling large empty-space skips in a single step.
- **Sequential / scan** — iterating all occupied voxels for a simulation step — favors compact, cache-coherent layouts. Dense arrays and VDB's leaf-node bit masks both do well. Pointer-chasing trees do not.
- **Streaming** — loading chunks from disk as the camera moves — favors chunked hierarchies with LOD so only visible, appropriately detailed chunks need to be resident.

### does the structure need to live on the GPU?

GPUs hate pointer-chasing. A tree where each traversal step follows a pointer to a dynamically allocated node kills GPU parallelism because every thread takes a different path through memory. Structures that win on GPU share one trait: <em>flat, linear memory layout</em> that the whole warp can address coherently.

- **GPU-native**: flat dense arrays, NanoVDB (linearized tree written to a contiguous buffer), and hash/brick maps (fixed-size brick pool indexed by a hash table).
- **GPU-capable with effort**: SVO and SVDAG require custom GPU traversal kernels and careful memory packing, but they can run well once the kernel is written.
- **CPU-first**: OpenVDB's dynamic pointer-based tree is designed for CPU; it needs conversion to NanoVDB before going on GPU.

### do you need level of detail?

LOD matters when the scene is large and you need to render it at multiple distances, or when you want to stream in progressively finer data.

- **Native LOD**: VDB (tile values aggregate child contents), chunked hierarchies (Aokana, Godot Terrain combine eight LOD-n chunks into one LOD-(n+1) chunk), and mipmapped 3D textures (used by VXGI and similar GI systems).
- **No native LOD**: a flat SVDAG or SVO has one resolution; LOD requires separate copies at each scale, or a carefully engineered multi-resolution variant.

### how rich does each voxel's payload need to be?

Some applications need a single occupancy bit. Others need color, material ID, SDF value, velocity, and temperature — simultaneously.

- **Single-channel or binary**: SVDAG is optimal — it was designed to compress binary occupancy geometry.
- **Fixed, small payload** (color + material): chunked dense and hash/brick maps extend trivially by widening the cell struct.
- **Multiple named, typed channels** (density, temperature, velocity, emission): VDB is purpose-built for this. Each channel is a separate grid sharing the same sparse topology; you compose them at read time.
- **High-precision contour / SDF**: Laine & Karras's ESVO stores per-voxel contour data alongside occupancy [1], making it suitable when you need sub-voxel surface precision without a full mesh.

---

## the structures at a glance

The table below scores each structure on the six axes. The scores are relative — "high" means the structure handles this well with no special effort; "low" means it fights you or needs significant engineering around it.

| | [dense / chunked](./dense-grids-and-chunks.md) | [SVO](./octrees-and-svo.md) | [SVDAG](./sparse-voxel-dags.md) | [hash / brick](./hash-grids-and-bricks.md) | [VDB / NanoVDB](./openvdb-and-nanovdb.md) |
|---|---|---|---|---|---|
| **sparsity handling** | low (chunked: medium) | high | high | high | high |
| **mutability** | high | medium | low | high | high (VDB) / none (NanoVDB) |
| **random access** | high | medium | medium | high | medium |
| **ray traversal** | low | high | high | medium | high |
| **GPU residency** | high | medium | medium | high | low (VDB) / high (NanoVDB) |
| **native LOD** | low (chunked: high) | medium | low | low | high |
| **attribute richness** | medium | low–medium | low (binary native) | medium | high |
| **compression** | low (with RLE/palette: medium) | medium | very high | low | medium |

---

## use X instead of Y because Z — per use case

### block-building game

<em>Use chunked dense arrays</em> instead of an octree because player edits land at arbitrary positions in real time, and writing to a flat array is an indexed store — no traversal, no rebalancing, no pointer fixup. A 16³ or 32³ chunk fits in cache; a column of chunks streams off disk as the player moves. Palette compression and run-length encoding cut memory within each chunk for mostly-uniform regions.

SVO or DAG would give you better compression in empty regions but terrible edit throughput — every write risks invalidating a large branch of the tree. Hash grids are a reasonable alternative if your world is extremely large and irregular, but most block-game engines (Minecraft, Godot Terrain, and open-source equivalents) converge on chunked dense because the edit path is trivially simple.

**When to add a DAG layer**: if you need a static, compressed, streamable snapshot of the world for distant LOD rendering — Aokana (2025) demonstrates exactly this: editable chunked data locally, SVDAG-compressed chunks for distant streaming [2].

### medical and scientific volume

<em>Use VDB (or a brick-pyramid)</em> instead of a single dense grid because medical volumes — a CT scan of a chest, an MRI of a brain — are dominated by air and tissue boundaries. A 512³ CT volume at 16-bit density is 268 MB as a dense array; as a sparse structure, only the voxels with non-trivial density values need storage.

More importantly, clinicians need multi-resolution access: a full-body survey at coarse resolution, then zoom into a region of interest at full resolution. VDB's hierarchical tiles naturally encode this. NanoVDB can carry a prepped volume to GPU for interactive direct volume rendering.

A pure dense array is reasonable only when the scan is small and entirely filled (e.g., a segmented organ extracted from a larger scan), since the indexing simplicity matters for downstream segmentation and simulation algorithms.

Attribute richness matters here: a single VDB grid carries the raw Hounsfield values; companion grids can carry segmentation labels, gradient vectors, or simulated flow fields — all sparse, all sharing the same spatial topology.

### VFX smoke, fire, and clouds

<em>Use OpenVDB for simulation and NanoVDB for GPU rendering</em> instead of a dense grid, because VFX volumes are maximally sparse — a plume of smoke occupies perhaps 2–5 % of its bounding box — and they store several named channels simultaneously (density, temperature, velocity, fuel). VDB was designed for exactly this workflow [3].

The pipeline is now industry-standard: Houdini's Pyro solver writes `.vdb` files per frame (each frame is a snapshot with dynamic topology), downstream compositing or render engines load those files, convert to NanoVDB for GPU [4], and ray-march through the volume. Arnold, Blender Cycles, and NVIDIA Omniverse all follow this path.

Dense 3D textures appear in real-time game VFX (where a baked, loopable cloud texture is acceptable), but for film-quality offline simulation you need VDB's dynamic topology — the shape of the smoke changes every frame in ways you can't pre-bake.

### voxel global illumination

<em>Use a dense 3D texture with mipmaps</em> instead of a sparse tree, because voxel cone tracing works by ray-marching through the mipmap hierarchy and sampling at the cone's footprint radius. The math assumes uniform spacing and instant neighbor access — qualities a regular grid provides and a pointer-chasing tree does not [5].

NVIDIA's VXGI, Crassin et al.'s original voxel cone tracing implementation, and Wicked Engine all voxelize the scene into a fixed-resolution dense 3D texture (typically 128³ to 512³) each frame. The mipmap chain encodes the average emittance/opacity over progressively coarser volumes, which is exactly what a cone trace needs to accumulate as it widens.

The tradeoff: the fixed resolution limits how fine the GI can be, and the dense 3D texture is expensive at high resolution. For larger scenes, clip-mapped voxel grids (a fixed window of dense voxels that follows the camera) keep memory bounded while covering a useful GI range. Hash grids (as used in Lumen's surface cache and similar systems) are an alternative for dynamic scenes where the voxel topology changes frame to frame.

### robotics mapping and reconstruction

<em>Use an octree (OctoMap / SVO)</em> instead of a dense occupancy grid because sensor data from LiDAR and depth cameras is sparse — most of the environment is empty space that the sensor never returned a measurement for. Storing that emptiness explicitly in a dense grid wastes memory that embedded systems can't spare.

OctoMap [6] stores a probabilistic occupancy value at each occupied node and compresses subtrees where all children share the same value. This handles the three-way state a robot needs: occupied, free, and unknown — the last being the default for unseen cells. Updates arrive as sensor rays; the octree supports efficient ray insertion for marking free space along the ray and occupied at the hit point.

The O(log n) write cost per sensor return is acceptable at typical sensor rates (tens of thousands of returns per second). Dense grids would be faster per-write but require knowing the map bounds in advance and pre-allocating for the full environment — impractical for exploration tasks.

For reconstruction pipelines that output to a mesh, TSDF fusion (truncated signed distance fields, as in KinectFusion) often uses a hash grid of TSDF bricks rather than an octree, because the TSDF needs constant-time insertion for streaming camera input [7].

### raytraced high-detail static scene

<em>Use SVDAG</em> instead of SVO because merging identical subtrees yields very high compression — scenes with geometric repetition (architecture, forests, rocks) can reach 10 voxels per bit [8] — and the resulting structure is small enough to fit entirely on GPU. A raytracer that works from the SVDAG traverses it exactly like an SVO (the branching logic is identical) but pays far less in memory bandwidth.

For open-world scales, Aokana (2025) combines per-chunk SVDAGs with a streaming LOD system that keeps only about 5 % of scene data resident in VRAM at any time, rendering tens of billions of voxels at ~6 ms per frame [2].

The constraint is immutability: once the SVDAG is built, editing a single voxel may require rebuilding many shared nodes. For scenes that never change after baking, this is no cost at all.

---

## combinations that appear in real systems

No rule says you must pick one structure for the whole pipeline. The most practical real-world systems separate concerns:

**editable layer + compressed render layer**

Keep the live, editable representation as chunked dense arrays (fast writes, simple code). When preparing for rendering — especially for distant or static content — compress those chunks into SVDAG or VDB. Aokana (2025) does this explicitly: player-proximate chunks remain mutable; chunks that have moved into the distance are compressed and streamed as SVDAGs [2]. The two representations don't need to stay in sync in real time — only when content transitions from active to static.

**simulation layer + render layer**

OpenVDB for CPU simulation (dynamic topology, multi-channel, mutable), converted per-frame to NanoVDB for GPU rendering. The conversion is cheap relative to the simulation step and the render step, and each layer gets the layout it needs [3][4].

**GI voxel cache + geometry store**

In a full rendering pipeline, the geometry store (chunked dense or SVDAG) and the GI voxel cache (dense 3D texture, possibly clip-mapped) are separate structures serving separate consumers. The renderer re-voxelizes the visible scene into the GI texture each frame; the geometry store is never touched by the GI system.

---

## putting it together with the rest of the pipeline

The store you pick constrains what comes after it.

- **Meshing**: chunked dense arrays work naturally with greedy meshing and marching cubes; SVO/DAG require extracting an isosurface from the tree, which is less direct. See [choosing a meshing algorithm](../meshing/choosing-a-meshing-algorithm.md) for how storage layout influences the meshing pass.
- **Rendering**: a DAG or dense grid feeds a very different render path than a VDB volume. See [choosing a render path](../rendering/choosing-a-render-path.md) for how the store connects to rasterization vs. ray-marching vs. SDF rendering.

The [storage problem](./the-storage-problem.md) page frames the O(n³) pressure that makes this choice matter in the first place — worth reading if you haven't.

---

## quick reference

| if your project is... | reach for... | avoid... |
|---|---|---|
| block-building / editable terrain | chunked dense + palette compression | SVDAG (rebuild cost on edit) |
| medical / scientific volume | VDB (+ NanoVDB for GPU) | dense at full scan resolution (memory) |
| VFX offline simulation | OpenVDB → NanoVDB pipeline | dense 3D texture (no multi-channel, no dynamic topology) |
| voxel GI / cone tracing | dense 3D texture with mipmaps | deep trees (cache incoherence kills throughput) |
| robotics / SLAM mapping | SVO / OctoMap | dense grid (unknown map bounds, wasted memory) |
| raytraced static scene | SVDAG (per-chunk if large) | SVO (same traversal, worse compression) |
| large open-world (any edit level) | chunked dense (hot) + SVDAG (cold) | one structure for everything |

---

## references

[1] Laine, S. and Karras, T. (2010). "Efficient Sparse Voxel Octrees." *Proceedings of the 2010 ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games* (I3D '10). DOI: 10.1145/1730804.1730814. [local PDF](../papers/laine-karras-2010-efficient-sparse-voxel-octrees.pdf) · [source](https://research.nvidia.com/sites/default/files/pubs/2010-02_Efficient-Sparse-Voxel/laine2010i3d_paper.pdf)

[2] Fang, Y. et al. (2025). "Aokana: A GPU-Driven Voxel Rendering Framework for Open World Games." *Proceedings of the ACM on Computer Graphics and Interactive Techniques* (HPG 2025). DOI: 10.1145/3728299. [local PDF](../papers/fang-2025-aokana-gpu-driven-voxel-rendering.pdf) · [source](https://arxiv.org/abs/2505.02017)

[3] Museth, K. (2013). "VDB: High-Resolution Sparse Volumes with Dynamic Topology." *ACM Transactions on Graphics*, 32(3), Article 27. DOI: 10.1145/2487228.2487235. [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf) · [source](https://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf)

[4] Museth, K. (2021). "NanoVDB: A GPU-Friendly and Portable VDB Data Structure For Real-Time Rendering And Simulation." *ACM SIGGRAPH 2021 Talks*. DOI: 10.1145/3450623.3464653. [local PDF](../papers/museth-2021-nanovdb-gpu-friendly-portable-vdb.pdf) · [source](https://dl.acm.org/doi/10.1145/3450623.3464653)

[5] Crassin, C., Neyret, F., Sainz, M., Green, S., and Eisemann, E. (2011). "Interactive Indirect Illumination Using Voxel Cone Tracing." *Computer Graphics Forum*, 30(7), 1921–1930. DOI: 10.1111/j.1467-8659.2011.02063.x. [local PDF](../papers/crassin-2011-voxel-cone-tracing.pdf) · [source](https://research.nvidia.com/publication/2011-09_interactive-indirect-illumination-using-voxel-cone-tracing)

[6] Hornung, A., Wurm, K. M., Bennewitz, M., Stachniss, C., and Burgard, W. (2013). "OctoMap: An Efficient Probabilistic 3D Mapping Framework Based on Octrees." *Autonomous Robots*, 34(3), 189–206. DOI: 10.1007/s10514-012-9321-0. [local PDF](../papers/elfes-1990-occupancy-grids-stochastic-spatial-representation.pdf) · [source](https://octomap.github.io/)

[7] Arbore, G., et al. (2024). "Hybrid Voxel Formats for Efficient Ray Tracing." *arXiv preprint*. arXiv: 2410.14128. [local PDF](../papers/arbore-2024-hybrid-voxel-formats-ray-tracing.pdf) · [source](https://arxiv.org/abs/2410.14128)

[8] Kämpe, V., Sintorn, E., and Assarsson, U. (2013). "High Resolution Sparse Voxel DAGs." *ACM Transactions on Graphics*, 32(4), Article 101. DOI: 10.1145/2461912.2462024. [local PDF](../papers/kampe-sintorn-assarsson-2013-high-resolution-sparse-voxel-dags.pdf) · [source](https://dl.acm.org/doi/10.1145/2461912.2462024)
