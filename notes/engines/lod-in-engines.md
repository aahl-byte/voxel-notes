<link rel="stylesheet" href="./css/globals.css">

# lod in engines

Imagine a game world that stretches for kilometers in every direction — mountains, valleys, settlements, sky. Rendering every distant chunk at full resolution is impossible: the triangle count would be crushing and the memory budget would overflow before the horizon was halfway filled. The goal is to make near terrain look rich and detailed while far terrain still reads correctly — cliff shapes, ridgelines, plains — even though it's built from far fewer polygons and sampler voxels.

The technique that makes this possible works as follows: keep a full-resolution representation of the world close to the camera, and progressively store and sample coarser voxels the further a region lies from the viewer. Each step away from the camera doubles the voxel size (and halves resolution), so the triangle and memory cost grows very slowly as view distance increases — logarithmically, not cubically. That idea, applied at the whole-engine level, is called <em>level of detail (LOD)</em>.

This page is about the engine side of that system — how to organize world data into LOD tiers, how to decide each frame which tier a chunk belongs in, and what goes wrong at the boundaries. The meshing side — how to stitch a crack-free mesh across a resolution boundary — is covered in [lod seams and transvoxel](../meshing/lod-seams-and-transvoxel.md).

---

## the core idea — coarser voxels with distance

A full-resolution chunk might hold 32×32×32 voxels at a cell size of 0.5 m. That same patch of world at LOD 1 is still 32×32×32 cells, but each cell is now 1 m wide — the chunk covers twice as much territory with the same voxel budget. At LOD 2, each cell is 2 m wide; at LOD N, 2^N × 0.5 m wide.

The key insight: **the number of voxels per chunk stays roughly constant; the physical scale of each voxel doubles with each LOD step.** This means:

- Memory per chunk is nearly constant regardless of LOD.
- Triangle count per chunk is nearly constant (the mesh complexity scales with voxel count, not world size).
- Total chunks on screen at any one time is bounded — a sphere of radius R, filled with LOD-0 chunks near the center and progressively coarser chunks toward the edge, has O(R / chunk_size) layers and a bounded total chunk count.

The view distance can therefore be enormous — tens or hundreds of chunk lengths — while the GPU sees a roughly constant workload each frame.

---

## three ways to organize LOD tiers

There are three common schemes for partitioning the world into LOD regions. They differ in how they draw the boundary between detail levels and how they update when the camera moves.

### distance rings

The simplest scheme places the camera at the center and draws concentric shells around it. LOD 0 fills the innermost shell (say, within 128 m), LOD 1 fills the next ring (128–256 m), LOD 2 fills 256–512 m, and so on. Each shell holds a fixed set of fixed-size chunks at a fixed resolution.

- **what it is:** a set of concentric spherical (or cubic) bands, each at a fixed LOD.
- **when to use it:** flat or heightmap-style voxel worlds, simple implementations, block-style games where abrupt resolution changes are acceptable.
- **strength:** trivially easy to implement and reason about — the LOD of any chunk is determined by one distance comparison.
- **weakness:** the boundary between rings is a hard line; every chunk in a ring is the same LOD regardless of how interesting the terrain is at that spot.

### octree LOD

An <em>octree LOD</em> stores the world in a hierarchical tree where each node covers a cubic region and can be subdivided into eight children. To assign LODs, the engine traverses the tree starting from the root and subdivides any node whose covered region is close enough to the camera to warrant more detail. Recursion continues until a node is either far enough away that its current resolution is sufficient, or it can no longer be subdivided. The result is a tree whose leaves are finer (higher LOD) near the camera and coarser farther away.

- **what it is:** a recursive spatial subdivision where the subdivision depth at any location is proportional to camera proximity.
- **when to use it:** smooth voxel terrain, large worlds, any case where you want a gradual and spatially adaptive transition between detail levels rather than hard rings.
- **strength:** naturally handles irregular geometry — dense urban areas near the player, empty sky above all get the depth they warrant. Memory is spent where it matters. The tree also directly gives you the chunks to mesh (each leaf is one mesh unit) and makes neighbor queries straightforward.
- **weakness:** tree structure must be updated when the camera moves, and rebuilding affected subtrees costs CPU time. Chunk boundaries don't align to a regular grid, which makes some streaming strategies harder.

The Voxel Plugin (Unreal) and Godot's `VoxelLodTerrain` both use this approach: each LOD level holds blocks twice the voxel size of the level below it, and the tree is rebuilt when the camera crosses an invoker distance threshold [1][2]. A concrete example: LOD 0 blocks are 32×32×32 voxels at 0.5 m each; LOD 1 blocks are 32×32×32 voxels at 1 m each — same block count, twice the world coverage per block.

### clipmap

A <em>clipmap</em> (or geometry clipmap) stores the world as a set of nested regular grids, each centered on the camera. The innermost grid is dense (fine resolution); each successive outer grid covers twice as much area in each axis at half the resolution. As the camera moves, the grids scroll — only the newly visible strip needs to be refilled, making incremental update extremely cheap.

Losasso and Hoppe introduced geometry clipmaps for heightmap terrain at SIGGRAPH 2004 and demonstrated rendering a 40 GB dataset of US terrain interactively [3]. The concept extends naturally to 3D voxel terrain: each grid level in the clipmap corresponds to one LOD, and the grids are stacked in 3D rather than 2D.

- **what it is:** nested regular grids centered on the camera, each at half the resolution of the grid inside it, scrolled incrementally as the camera moves.
- **when to use it:** large open-world heightmap-like terrain where the camera moves continuously; situations where you want predictable, regular data layout on the GPU.
- **strength:** the regular grid layout is GPU-friendly — vertex buffers are dense, update patterns are predictable, and the bandwidth cost per frame is minimal because only a thin strip of each level changes per frame.
- **weakness:** the 3D extension of clipmaps is more complex than the 2D heightmap version; managing six faces of an axis-aligned box as the camera moves adds implementation overhead. Godot's `VoxelLodTerrain` calls the clipbox variant its "clipbox" streaming mode, supporting multiple viewers [2].

---

## ray-traced engines — lod for free

Rasterization-based engines need to explicitly manage LOD because they produce a mesh for each chunk and submit it to the rasterizer. Ray-traced voxel engines work differently — and get LOD largely for free from the data structure itself.

A sparse voxel octree (SVO) as described in [octrees and SVOs](../storing/octrees-and-svo.md) already encodes a hierarchy: the root covers the whole world coarsely, and each level of children refines that coverage. A ray traversing the tree (as in [SVO ray tracing](../rendering/sparse-voxel-octree-raytracing.md)) descends to finer nodes only when the ray is close enough and the voxel would otherwise subtend more than one screen pixel. When a node is small enough on screen, the ray stops and shades that node's pre-filtered appearance data without descending further.

GigaVoxels (Crassin et al., 2009) is the canonical example of this idea: a ray-guided streaming system where the octree nodes accessed during rendering drive what data is loaded next [4]. The LOD of any region is the depth reached by rays that pass through it — close objects generate deep traversals (fine detail), distant objects generate shallow traversals (coarse detail). No explicit LOD tier assignment happens; it falls out of the geometry of the tree and the size of the pixel on screen.

This is one of the strongest arguments for SVO-based ray tracing: it eliminates the explicit LOD scheduling problem entirely, replacing it with an implicit screen-space coverage test.

---

## two artifacts and their fixes

Changing resolution at a boundary creates two distinct visual problems. Understanding them separately is important because they have different causes and different fixes.

### cracks

When a fine-resolution chunk and a coarse-resolution chunk share a face, their meshes don't agree on vertex positions along that edge. The fine side has more vertices; the coarse side has fewer. Gaps appear — <em>cracks</em> — that expose the interior of the terrain.

The fix is Transvoxel (Lengyel, 2010): a set of special transition cells that are placed exactly at LOD boundaries [5]. Each transition cell bridges the mismatch — it knows the fine resolution on one side and the coarse resolution on the other and produces a triangle patch that fills the gap without overlapping either mesh. The algorithm defines 73 equivalence classes for transition cell configurations (out of 512 possible corner combinations), analogous to the 15 equivalence classes of standard marching cubes.

This is meshing work, not engine scheduling work — the engine must know which chunks sit at an LOD boundary and request both a regular mesh and a transition mesh for those chunks. See [lod seams and transvoxel](../meshing/lod-seams-and-transvoxel.md) for the full treatment of how transition cells are constructed.

### popping

When a chunk swaps from one LOD to another, its mesh changes shape abruptly. The player sees a ridge suddenly sharpen or a distant hill change silhouette — <em>popping</em>. Cracks are a geometric problem; popping is a perceptual problem.

There are three main mitigation strategies:

**Geomorphing.** Instead of swapping the mesh instantly, the engine holds both the old and new vertex positions and interpolates between them over several frames. A vertex at the coarse LOD position smoothly slides to its fine LOD position over a short window (typically 100–300 ms). The 0fps blog demonstrates this for blocky voxels using a quantization function `L_i(v) = 2^i * floor(v / 2^i)` applied in the vertex shader — at LOD `i`, vertices snap to coarser grid positions and then morph back as detail is restored [6]. Geomorphing adds no extra draw calls and is imperceptible at normal play speed.

**Dithered / temporal blend.** The incoming and outgoing LOD meshes are rendered simultaneously, with the outgoing mesh rendered at decreasing opacity using a screen-door (dithered) pattern. When combined with temporal anti-aliasing (TAA), which accumulates and blends samples across frames using per-frame subpixel jitter, the dithered pixels average out into a smooth dissolve over several frames. Unreal Engine uses this approach — the `DitherTemporalAA` node samples a tileable noise texture in screen space, producing per-pixel LOD selection that TAA then smooths into a continuous transition [7]. The cost: both meshes are drawn simultaneously, roughly doubling the workload for the duration of the transition.

**Accept it for distant chunks.** At sufficient distance, the subtended solid angle of the changed region is small enough that the eye doesn't notice. Many engines only apply geomorphing or blending within a certain radius and let distant LOD swaps go unmitigated.

---

## scheduling: choosing each chunk's lod per frame

Every frame the engine must answer: for each chunk in the view frustum, what LOD does it get?

The naive answer — compute camera distance, pick the appropriate LOD ring — is correct but naively applied it causes problems. If the camera sits exactly at a ring boundary, chunks near that boundary oscillate between two LODs every frame, triggering continuous mesh rebuilds and popping.

**Hysteresis** solves this. The rule becomes: a chunk upgrades to a finer LOD only when distance drops below threshold `T_in`, but it downgrades to a coarser LOD only when distance exceeds `T_out > T_in`. The gap between `T_in` and `T_out` is the hysteresis band. A chunk sitting at a boundary stays at its current LOD until the camera has moved clearly past the threshold in one direction. This is the same principle used in thermostats and electrical comparators: add a deadband to prevent oscillation [8].

**Per-frame mesh budget.** Building a mesh takes time. If every chunk requesting a new LOD is rebuilt in a single frame, the frame rate collapses. The engine caps how many chunk rebuilds happen per frame — Godot's `VoxelLodTerrain` exposes `voxel/threads/main/time_budget_ms` (default ~8 ms) which limits the CPU time spent on voxel tasks per frame [2]. Chunks that need rebuilding enter a priority queue sorted by distance; the closest ones rebuild first. Distant requests wait in queue and are served over subsequent frames.

**LRU eviction.** When the memory pool for loaded chunks fills up, the engine must evict something. A least-recently-used (LRU) policy evicts the chunk that was last accessed furthest in the past — typically a distant coarse-LOD chunk that the camera has moved away from. If the camera reverses direction, that chunk must be re-streamed, which is why a streaming prefetch margin (loading slightly beyond the current view distance) is worth the memory cost. The [chunk management and streaming](./chunk-management-and-streaming.md) page covers the streaming side of this budget in full.

**Screen-space error.** A more principled alternative to pure distance: compute the projected screen-space size of the detail difference between two LODs, and upgrade only if that difference exceeds one pixel. This is the approach used in geometry clipmaps and in the `procworld` error-metric system [9]. Content-aware error metrics allow a flat plain to stay coarse even if it's nearby, while a detailed cliff gets finer LOD even at distance — because the flat plain's detail difference between LOD levels is zero regardless of distance. This connects to the broader [performance budget](../optimization/the-performance-budget.md) and the [LOD and culling](../optimization/lod-and-culling.md) techniques that tune triangle throughput against frame time targets.

---

## putting the pieces together

A typical LOD pipeline for a smooth voxel terrain engine looks like this:

1. **Each frame:** traverse the LOD octree (or clipmap) from the camera outward. For each leaf chunk, apply hysteresis to determine its target LOD.
2. **Priority queue:** chunks whose target LOD differs from their current LOD are enqueued for rebuild, sorted by camera distance.
3. **Worker threads:** background threads pull from the queue and generate meshes (and transition meshes for boundary chunks) up to the per-frame time budget. See [threading and meshing pipeline](./threading-and-meshing-pipeline.md) for how this parallelism is structured.
4. **Upload:** finished meshes are uploaded to the GPU. The old mesh stays visible until the new one is ready — this prevents a hole appearing during the transition.
5. **Render:** each chunk submits its mesh. Chunks at LOD boundaries also submit their Transvoxel transition mesh to fill cracks. Chunks in transition play their geomorphing or dithered blend.
6. **Eviction:** if the chunk pool is full, LRU chunks are evicted. A prefetch margin loads chunks one or two rings beyond the current view distance so evicted chunks can be re-streamed before they become visible again.

The [anatomy of a voxel engine](./anatomy-of-a-voxel-engine.md) page shows where this LOD loop fits in the broader engine architecture — next to the chunk generation, physics, and edit systems.

---

## contrast: which scheme to choose

| scheme | data layout | update cost when camera moves | best for |
|---|---|---|---|
| distance rings | fixed grid per ring | rebuild entire ring boundary | simple block games, flat worlds |
| octree LOD | tree of variable-depth nodes | rebuild affected subtrees only | smooth terrain, adaptive detail, large worlds |
| clipmap | nested scrolling grids | scroll thin strip per grid level | heightmap-like terrain, GPU-friendly layout, multi-km view distances |
| SVO ray tracing | single tree, no explicit chunks | no LOD schedule needed — implicit | path-traced/ray-traced voxel renderers |

The octree and clipmap approaches are not mutually exclusive — some engines use a clipmap for the outer LOD levels (where regular layout matters most) and switch to an octree near the player (where adaptive detail matters most).

---

## references

[1] Voxel Plugin Documentation. "World Size and Level Of Details." Voxel Plugin 1.2. https://docs.voxelplugin.com/1.2/core-systems/voxelworld/world-size-and-level-of-details (Retrieved: 2026-06-16)

[2] Zylann. "VoxelLodTerrain — Voxel Tools Documentation." Read the Docs. https://voxel-tools.readthedocs.io/en/latest/api/VoxelLodTerrain/ and https://voxel-tools.readthedocs.io/en/latest/smooth_terrain/ (Retrieved: 2026-06-16)

[3] Losasso, F. and Hoppe, H. (2004). "Geometry Clipmaps: Terrain Rendering Using Nested Regular Grids." *ACM Transactions on Graphics (SIGGRAPH)*, 23(3). DOI: 10.1145/1015706.1015799. [local PDF](../papers/losasso-hoppe-2004-geometry-clipmaps.pdf) · [source](https://hhoppe.com/geomclipmap.pdf)

[4] Crassin, C., Neyret, F., Lefebvre, S., and Eisemann, E. (2009). "GigaVoxels: Ray-Guided Streaming for Efficient and Detailed Voxel Rendering." *ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (I3D)*. DOI: 10.1145/1507149.1507152. [local PDF](../papers/crassin-2009-gigavoxels-ray-guided-streaming.pdf) · [source](https://maverick.inria.fr/Publications/2009/CNLE09/CNLE09.pdf)

[5] Lengyel, E. (2010). "Voxel-Based Terrain for Real-Time Virtual Simulations." PhD dissertation, University of California, Davis. (Transvoxel algorithm.) [local PDF](../papers/lengyel-2010-voxel-terrain-transvoxel.pdf) · [source](https://transvoxel.org/Lengyel-VoxelTerrain.pdf)

[6] Kaminsky, M. (2018). "A Level of Detail Method for Blocky Voxels." *0fps blog*. https://0fps.net/2018/03/03/a-level-of-detail-method-for-blocky-voxels/ (Retrieved: 2026-06-16)

[7] Cesium. (2022). "Smoother LOD Transitions in Cesium for Unreal with Dithered Opacity Masking." *Cesium Blog*. https://cesium.com/blog/2022/10/20/smoother-lod-transitions-in-cesium-for-unreal/ (Retrieved: 2026-06-16)

[8] Jackson, D. (2018). "Using Quadtrees for Level-of-Detail in Voxel Generation." *Medium*. https://medium.com/@danieljackson97123/using-quadtrees-for-level-of-detail-in-voxel-generation-517f98f3bf50 (Retrieved: 2026-06-16)

[9] Procworld. (2016). "Improved LOD." *Procedural World blog*. http://procworld.blogspot.com/2016/11/improved-lod.html (Retrieved: 2026-06-16)

[10] Laine, S. and Karras, T. (2010). "Efficient Sparse Voxel Octrees." *ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (I3D)*. DOI: 10.1145/1730804.1730814. [local PDF](../papers/laine-karras-2010-efficient-sparse-voxel-octrees.pdf) · [source](https://research.nvidia.com/publication/2010-02_efficient-sparse-voxel-octrees)

[11] Fang, Y., Wang, Q., and Wang, W. (2025). "Aokana: A GPU-Driven Voxel Rendering Framework for Open World Games." *arXiv preprint*. arXiv:2505.02017. [local PDF](../papers/fang-2025-aokana-gpu-driven-voxel-rendering.pdf) · [source](https://arxiv.org/abs/2505.02017)
