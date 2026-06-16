<link rel="stylesheet" href="./css/globals.css">

# choosing a render path

You have voxel data. Something needs to appear on screen. That sentence hides a decision with real consequences: the rendering path you choose determines what hardware you can target, whether players can edit the world at runtime, how the scene scales from a single room to an open world, and how cleanly this system sits alongside the rest of an existing engine. Make the wrong choice and you are either re-architecting the renderer mid-project or shipping a game that runs at 15 fps on the target hardware.

This page is a decision guide. It assumes you have read [ways to render voxels](./ways-to-render-voxels.md) for the plain-language shape of each path, and that you are now trying to pick one. The paths are mesh rasterization, uniform-grid ray marching, SVO/DAG ray tracing, volume ray casting, and splatting. This page explains the axes of the decision, scores each path against them, and gives direct "use X instead of Y because Z" picks for each major use case.

The render path is coupled to the data model — read [choosing a voxel store](../storing/choosing-a-voxel-store.md) first if you have not already. And if your path goes through a mesh, the choice of meshing algorithm has its own set of trade-offs: [choosing a meshing algorithm](../meshing/choosing-a-meshing-algorithm.md).

---

## the five paths, briefly

Before comparing them, a one-line description of each. These are described more fully in the dedicated pages below; here they are named so the table headings make sense.

- <em>Mesh rasterization</em> — extract a triangle mesh from the voxel data (via greedy meshing, marching cubes, surface nets, dual contouring) then feed it to the GPU's fixed-function rasterizer as ordinary geometry. This is [why mesh voxels](../meshing/why-mesh-voxels.md) exists as a decision in the first place.
- <em>Uniform-grid ray marching</em> — shoot a ray per pixel and step it through a regular voxel grid cell by cell using the DDA algorithm until it hits an occupied cell. See [grid ray traversal](./grid-ray-traversal.md) for the mechanics.
- <em>SVO/DAG ray tracing</em> — shoot a ray per pixel through a hierarchical sparse voxel octree or DAG, skipping large empty subtrees in a single step. See [sparse voxel octree raytracing](./sparse-voxel-octree-raytracing.md).
- <em>Volume ray casting</em> — shoot a ray per pixel and accumulate color and opacity along its length, without stopping at the first hit. This produces translucent volume images rather than opaque surfaces. See [volume ray casting](./volume-ray-casting.md).
- <em>Splatting</em> — project each occupied voxel (or voxel cluster) directly into screen space as a small polygon or Gaussian ellipse, without tracing rays. See [splatting and point rendering](./splatting-and-point-rendering.md).

---

## the decision axes

Five questions shape the choice. Answer these first; the table and use-case picks below are indexed to them.

### opaque surface or translucent volume?

The most fundamental split. If every visible voxel is either completely solid or completely empty — terrain blocks, a scanned object, a building — you want an <em>opaque surface path</em>: a ray stops at the first occupied voxel, or the rasterizer draws the front face of a mesh. Mesh rasterization, uniform-grid DDA, and SVO/DAG tracing all work this way.

If the voxels represent density that varies smoothly and light passes through — smoke, fog, cloud, tissue, flame — you need a <em>translucent volume path</em> that accumulates along the ray rather than stopping. Volume ray casting is the direct approach. Splatting can approximate it by rendering partially transparent splats in back-to-front order.

This axis immediately eliminates paths: you cannot use the standard rasterizer for a smoke volume without significant hacks, and volume ray casting is unnecessary overhead for a solid block world.

### static or dynamic/editable?

If the voxel data changes at runtime — player digs a tunnel, an explosion carves a crater, a simulation updates densities — the render path must either re-mesh or re-traverse a structure that can be updated cheaply.

- **Mesh rasterization** requires re-meshing the changed chunks and re-uploading geometry. At chunk granularity (16³ or 32³) this is fast enough for block games. The mesh itself is a derivative; the authoritative data is the voxel grid.
- **Uniform-grid DDA** reads the grid directly. A voxel edit is a write to the underlying array; the renderer sees the change on the next frame with no re-build step. This is the lowest latency option for highly dynamic data.
- **SVO/DAG tracing** requires rebuilding affected tree nodes on edit. SVO can be updated incrementally; DAG requires more extensive reconstruction when shared subtrees are invalidated. SVDAGs are optimally suited to scenes that are static after baking, and poorly suited to real-time edits.
- **Volume ray casting** reads the density field directly; an edit is a write to the field, similar to DDA. Works well for simulation outputs where the whole field is rewritten each frame anyway.
- **Splatting** requires rebuilding the splat list when the underlying data changes, though this is usually cheaper than full re-meshing.

### detail and scale

How many voxels are you rendering, and how small are they relative to a pixel?

At small scenes (a room, a single object), almost any path works. At large outdoor scale — millions to billions of voxels across an open world — you need level of detail and a path that can skip what is invisible or sub-pixel.

- **Mesh rasterization** with chunk-based LOD (swap high-res chunks for low-res meshes at distance) scales well, and GPU culling pipelines are mature.
- **SVO/DAG tracing** gains the most from scale because empty-subtree skipping means traversal cost grows with scene occupancy, not scene size. Aokana (2025) renders tens of billions of voxels in real time using a streaming SVDAG that keeps only ~5 % of scene data resident in VRAM [1].
- **Uniform-grid DDA** costs one step per voxel traversed along the ray; it does not skip empty regions efficiently beyond what a flat distance field can provide. At open-world scale without hierarchical structure it becomes untenable.
- **Volume ray casting** can be accelerated by empty-space skipping (pre-computed distance fields, VDB's sparse hierarchy), but it is fundamentally bounded by the number of non-empty samples accumulated per ray. Scales to large medical volumes, but not to open-world games.
- **Splatting** scales naturally — only the visible splats need to be submitted — but loses coherence when splat footprints become sub-pixel, producing aliasing.

### target hardware

Each path has a natural home in the GPU pipeline.

- **Mesh rasterization** uses the fixed-function rasterizer: vertex shaders, index buffers, depth testing. Every GPU since 2005 handles this efficiently. No special hardware required.
- **Uniform-grid DDA** and **SVO/DAG custom traversal** run as compute shaders or fragment shaders. They avoid the rasterizer entirely. They work on any GPU with compute capability, but cannot use RT cores (which only accelerate BVH traversal, not custom tree structures).
- **Hardware ray tracing (DXR / Vulkan RT / RT cores)** is designed for BVH-accelerated triangle intersection, not for voxel tree traversal. You can accelerate mesh rasterization's shadow or reflection passes with hardware RT, and you can splat voxels into a BVH to use RT acceleration, but direct SVO traversal does not benefit from RT cores.
- **Volume ray casting** runs as a compute or fragment shader. It benefits from GPU cache coherence (rays through neighboring pixels access nearby memory) but has no dedicated hardware acceleration.
- **3D Gaussian splatting** variants use the rasterizer in an unconventional mode: Gaussians are rasterized as screen-space splats. They fit naturally into a rasterizer pipeline.

### integration with an existing engine pipeline

A standalone voxel renderer is one thing; dropping voxel rendering into a mesh-based game engine is another. Existing engines (Unity, Unreal, Godot) are built around rasterized mesh geometry. Shadow maps, occlusion culling, LOD systems, post-processing, deferred shading — all of these assume geometry lives in a triangle mesh.

- **Mesh rasterization** integrates transparently. The meshed voxels enter the pipeline as ordinary draw calls. Shadow mapping, reflections, GI probes, and post-processing all work without changes. Aokana (2025) explicitly notes that its SVDAG renderer is designed to be integrated with mesh-based rendering, not to replace it [1].
- **Custom ray traversal** requires either a full deferred integration (ray-march into a G-buffer, composite with rasterized elements) or a complete renderer replacement. Either approach adds engineering complexity and breaks assumptions that existing tools rely on.
- **Volume ray casting** is best treated as a separate pass composited over the rasterized world — the standard approach in games that mix mesh geometry with volumetric smoke or fog.
- **Splatting** competes with the rasterizer for the same draw-call budget but can be composited similarly.

---

## comparison table

Each path scored on the five axes: surface/volume (S=surface, V=volume, B=both), mutability, scale/LOD, hardware requirement, and engine integration.

| | mesh rasterization | uniform-grid DDA | SVO/DAG tracing | volume ray casting | splatting |
|---|---|---|---|---|---|
| **opaque/translucent** | opaque | opaque | opaque | translucent | both |
| **handles runtime edits** | yes (re-mesh chunk) | yes (write to grid) | limited (DAG requires rebuild) | yes (write to field) | partial (rebuild splat list) |
| **scales to open world** | yes (chunk LOD) | no (no empty-skip) | yes (tree skipping) | no (dense sampling) | partial (splat culling) |
| **target hardware** | any GPU | compute shader | compute shader | compute shader | rasterizer or compute |
| **RT core benefit** | shadows/reflections | none | none (custom tree) | none | partial (BVH over splats) |
| **engine integration** | transparent | complex | complex | composited pass | composited pass |
| **best for** | game geometry | small dynamic scenes | massive static detail | translucent media | point clouds, NeRF |
| **worst for** | translucent volumes | large static scenes | real-time edits | opaque geometry | sharp-edged geometry |

---

## use X instead of Y because Z

### block-building / destructible game

<em>Use mesh rasterization</em> instead of SVO ray tracing because player edits need to reach the screen in under a frame. Re-meshing a 32³ chunk is a few milliseconds on CPU or a single compute dispatch; rebuilding a DAG that shares subtrees across the scene is not. The rasterizer is also what the rest of the engine — shadow maps, reflections, deferred shading, LOD — already expects.

SVO/DAG tracing would give you better compression and no meshing latency for static background content, but the edit path fights you at every turn. Use chunked dense storage as the edit layer; optionally compress distant, finalized chunks to SVDAG for streaming as Aokana does [1].

**When to consider DDA instead:** for a small-scale voxel sandbox (a single room, a game-jam scope) where you want the simplest possible renderer without any meshing pass. Grid DDA requires only a flat array and a ray loop.

### smooth procedural terrain

<em>Use mesh rasterization with marching cubes or dual contouring</em> instead of SVO tracing because smooth isosurface geometry (caves, overhangs, rolling hills) is the natural output of a signed-distance field, and that mesh integrates cleanly with existing normal mapping, shadow mapping, and material systems. The SDF-to-mesh conversion happens once per chunk and the result is a standard triangle mesh.

SVO tracing of a smooth SDF would require sub-voxel contour data (as in ESVO [2]) and a custom renderer — significant engineering for a result still coarser than a properly tuned marching cubes mesh. See [choosing a meshing algorithm](../meshing/choosing-a-meshing-algorithm.md) for the trade-offs within the meshing family.

**When to consider DDA ray marching instead:** if you want to skip meshing entirely and render the SDF directly by sphere-tracing (stepping by the SDF value rather than by a fixed cell width). This gives analytically smooth surfaces at any resolution but requires writing a custom surface shader and integrating it with shadows and GI manually.

### massive static detail

<em>Use SVO/DAG ray tracing</em> instead of mesh rasterization because the geometry is known in advance and DAG subtree merging can compress it to a fraction of its raw size — scenes with geometric repetition reach 10 voxels per bit [3] — and the resulting structure fits entirely on GPU. The traverse-and-shade loop produces correct depth, normal, and material for every pixel without generating and storing a triangle mesh.

Mesh rasterization would require generating, storing, and uploading a mesh for billions of voxels. The mesh can be larger than the raw voxel data for sparse, complex geometry; you lose the compression benefit and gain no visual improvement.

Hybrid formats — where different levels of the SVO hierarchy use different sub-structures (raw grid at the leaves, distance field mid-level, SVDAG at the top) — can push the performance/memory frontier beyond any single format [4].

**When to reintroduce meshing:** for LOD transitions at close range, where sub-voxel detail (beveled edges, smooth normals) matters more than memory compression. Aokana uses SVDAG for streaming but integrates with mesh rendering for detailed close-up content [1].

### medical / scientific volume

<em>Use volume ray casting</em> instead of surface extraction because the goal is to see *through* the data, not to see its boundary. A CT scan of a lung contains thousands of density values that a clinician needs to inspect simultaneously: bone, soft tissue, air, fluid. A transfer function maps each density to a color and opacity; the accumulation along the ray composites them into an image that reveals the interior at any angle and zoom level.

Extracting an isosurface (marching cubes at a threshold) would give you one surface — say, the bone — but hide everything inside and force the clinician to choose a threshold before they can look around. Volume ray casting lets the transfer function be adjusted interactively to emphasize different tissue types in the same render.

OpenVDB for CPU pre-processing and NanoVDB for GPU ray casting [5][6] is the practical pipeline for large volumes: the VDB sparse hierarchy skips air and empty space, focusing samples where density is non-trivial. GPU ray casting achieves interactive frame rates even at 512³ and above.

**When to add surface extraction:** for surgical planning where a precise 3D model of a specific structure (e.g., a tumor margin, a bone surface) needs to be measured, 3D-printed, or shared with other tools. Extract the isosurface for that one use; keep volume ray casting for exploration.

### VFX smoke, fire, and clouds

<em>Use volume ray casting over NanoVDB</em> instead of a dense 3D texture because VFX volumes are maximally sparse (a smoke plume may occupy 2–5 % of its bounding box) and change topology every frame. OpenVDB's dynamic sparse structure stores the non-trivial voxels per frame, carrying multiple named channels (density, temperature, velocity, emission) [5]. NanoVDB converts that to a GPU-friendly layout for the ray casting pass [6].

A baked dense 3D texture is a reasonable shortcut for a loopable real-time cloud or fog effect in a game, where full physical simulation is not needed. But for film-quality VFX where the simulation drives the shape, VDB's dynamic topology is non-negotiable — you cannot pre-allocate a dense grid for smoke that the solver grows and shrinks each frame.

Volume ray casting also composites naturally over a rasterized scene: render the mesh geometry into a depth buffer first, then ray cast the volume, stopping when the ray passes behind the depth buffer. This is standard practice in Houdini / Arnold / Blender Cycles pipelines.

### scanned point clouds / radiance fields

<em>Use splatting (3D Gaussian splatting or voxel splatting)</em> instead of mesh rasterization or ray tracing because scanned data — LiDAR, depth camera, NeRF capture — does not come with clean surface topology. Extracting a mesh from a noisy point cloud introduces reconstruction artifacts; ray tracing requires the data to be organized into a tree; splatting accepts each point (or cluster) as a self-contained splat without requiring connectivity.

3D Gaussian splatting represents each splat as a 3D Gaussian with per-splat opacity and spherical harmonics coefficients, rendering them as screen-space ellipses via the rasterizer [7]. The result is photorealistic novel-view synthesis at real-time rates, outperforming NeRF on speed by orders of magnitude while maintaining comparable quality. Sparse Voxels Rasterization (2025) extends this to pure voxel grids without Gaussians or neural networks, achieving comparable PSNR with a 10x FPS improvement over prior neural-free voxel methods [8].

The limitation of splatting is the absence of a clean surface: you cannot extract a mesh for collision, physics, or downstream processing directly from a Gaussian splat scene. For assets where a clean mesh matters, convert the Gaussians to a dense point cloud and run a surface reconstruction pass.

**When to use volume ray casting instead:** if the scan captures a translucent medium (tissue, subsurface scattering material) where the interior density variation needs to be visible rather than hidden behind a reconstructed surface.

---

## how data model and meshing lock in the render path

The render path is not a free choice — it is constrained by what the data model produces and what the meshing decision makes available.

- **Chunked dense arrays** produce data naturally readable by uniform-grid DDA and, after meshing, by the rasterizer. They do not produce an SVO without an explicit build step.
- **SVDAGs** are built for ray traversal. Running them through the rasterizer requires first extracting a mesh from the DAG — expensive and partially defeats the purpose.
- **VDB / NanoVDB** is purpose-built for volume ray casting and compute-shader traversal. It does not integrate with the hardware rasterizer directly.
- **Raw point clouds** are the native input to splatting and to surface reconstruction algorithms that might produce a mesh.

This coupling is why [choosing a voxel store](../storing/choosing-a-voxel-store.md) and [choosing a meshing algorithm](../meshing/choosing-a-meshing-algorithm.md) belong earlier in the pipeline than this page. The store you pick today constrains your renderer options before you write the first shader.

---

## quick reference

| if your project is... | reach for... | avoid... |
|---|---|---|
| block-building / destructible game | mesh rasterization (re-mesh per chunk) | DAG tracing (edit rebuild cost) |
| smooth procedural terrain | mesh rasterization + marching cubes / dual contouring | SVO tracing (custom surface shading overhead) |
| massive static detail, open world | SVO/DAG tracing (SVDAG per chunk, streamed) | uniform-grid DDA (no empty-skip at scale) |
| medical / scientific volume | volume ray casting (VDB + NanoVDB on GPU) | isosurface only (hides interior) |
| VFX smoke, fire, clouds | volume ray casting over NanoVDB | dense 3D texture (no dynamic topology) |
| scanned point clouds / radiance fields | splatting (3DGS or sparse voxel rasterization) | mesh rasterization (no clean topology) |
| hybrid: editable world + massive distant LOD | mesh (near) + SVDAG streaming (far) | one path for everything |

---

## references

[1] Fang, Y. et al. (2025). "Aokana: A GPU-Driven Voxel Rendering Framework for Open World Games." *Proceedings of the ACM on Computer Graphics and Interactive Techniques* (HPG / I3D 2025). DOI: 10.1145/3728299. [local PDF](../papers/fang-2025-aokana-gpu-driven-voxel-rendering.pdf) · [source](https://arxiv.org/abs/2505.02017)

[2] Laine, S. and Karras, T. (2010). "Efficient Sparse Voxel Octrees." *Proceedings of the 2010 ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games* (I3D '10). DOI: 10.1145/1730804.1730814. [local PDF](../papers/laine-karras-2010-efficient-sparse-voxel-octrees.pdf) · [source](https://research.nvidia.com/sites/default/files/pubs/2010-02_Efficient-Sparse-Voxel/laine2010i3d_paper.pdf)

[3] Kämpe, V., Sintorn, E., and Assarsson, U. (2013). "High Resolution Sparse Voxel DAGs." *ACM Transactions on Graphics*, 32(4), Article 101. DOI: 10.1145/2461912.2462024. [local PDF](../papers/kampe-sintorn-assarsson-2013-high-resolution-sparse-voxel-dags.pdf) · [source](https://dl.acm.org/doi/10.1145/2461912.2462024)

[4] Arbore, R. et al. (2024). "Hybrid Voxel Formats for Efficient Ray Tracing." *Advances in Visual Computing* (ISVC 2024). arXiv: 2410.14128. [local PDF](../papers/arbore-2024-hybrid-voxel-formats-ray-tracing.pdf) · [source](https://arxiv.org/abs/2410.14128)

[5] Museth, K. (2013). "VDB: High-Resolution Sparse Volumes with Dynamic Topology." *ACM Transactions on Graphics*, 32(3), Article 27. DOI: 10.1145/2487228.2487235. [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf) · [source](https://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf)

[6] Museth, K. (2021). "NanoVDB: A GPU-Friendly and Portable VDB Data Structure For Real-Time Rendering And Simulation." *ACM SIGGRAPH 2021 Talks*. DOI: 10.1145/3450623.3464653. [local PDF](../papers/museth-2021-nanovdb-gpu-friendly-portable-vdb.pdf) · [source](https://dl.acm.org/doi/10.1145/3450623.3464653)

[7] Kerbl, B., Kopanas, G., Leimkühler, T., and Drettakis, G. (2023). "3D Gaussian Splatting for Real-Time Radiance Field Rendering." *ACM Transactions on Graphics*, 42(4). DOI: 10.1145/3592433. [source](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/)

[8] Chen, Y. et al. (2024). "Sparse Voxels Rasterization: Real-Time High-Fidelity Radiance Field Rendering." arXiv: 2412.04459. [source](https://arxiv.org/abs/2412.04459)

[9] Crassin, C., Neyret, F., Lefebvre, S., and Eisemann, E. (2009). "GigaVoxels: Ray-Guided Streaming for Efficient and Detailed Voxel Rendering." *Proceedings of the 2009 ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games* (I3D '09). DOI: 10.1145/1507149.1507152. [local PDF](../papers/crassin-2009-gigavoxels-ray-guided-streaming.pdf) · [source](https://maverick.inria.fr/Publications/2009/CNLE09/)
