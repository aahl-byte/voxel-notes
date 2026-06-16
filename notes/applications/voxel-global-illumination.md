<link rel="stylesheet" href="./css/globals.css">

# voxel global illumination

A scene rendered with only direct light looks flat and unconvincing. Surfaces that face away from every light source are uniformly black; the underside of a table receives nothing from the ceiling lamp even though the floor around it is bouncing light in every direction. Real light doesn't stop at the first surface it hits — it scatters, reflects off walls, bleeds colour from a red carpet onto a white baseboard, and fills entire rooms with soft secondary illumination. Getting that bounced (indirect) light into a real-time frame has been one of the central problems in graphics for decades.

The traditional answers were either to precompute lightmaps — baking indirect light into textures offline, which breaks the moment anything moves — or to approximate it cheaply with ambient colour, which carries none of the shape or colour detail of real bounced light. Voxel global illumination offers a different path: it runs every frame, responds to moving objects and changing lights, and produces convincing soft indirect lighting and colour bleeding without any offline bake.

The key insight is to repurpose the voxel grid as a radiance cache. Take the scene's geometry and the direct light already hitting every surface, turn them into a 3D grid of small emitting cubes, build a hierarchy of progressively blurred versions of that grid, and then gather indirect light by sampling that hierarchy in a handful of widening queries per pixel. That gathering step — marching a cone through progressively coarser mip levels — is called <em>voxel cone tracing</em>, and it was introduced by Crassin et al. in 2011 [1]. The technique powered NVIDIA's VXGI middleware, the SVOGI system in early Unreal Engine 4, and CryEngine's SVOGI feature, and its lineage runs through to modern hybrid approaches like Unreal Engine 5's Lumen.

---

## the coarse picture

Think of indirect lighting as a two-question problem.

**Question 1:** What is every surface in the scene emitting (or reflecting) right now? This is the radiance of the scene — not just the light sources, but every surface that has already been hit by direct light and is now re-radiating energy.

**Question 2:** For a given pixel, how much of that scattered light arrives from each direction, and how much of it is blocked?

Answering question 1 with a voxel grid means [voxelizing the scene's geometry](../generating/mesh-voxelization.md) into a 3D texture and then computing direct lighting at each voxel cell, leaving each cell storing the outgoing radiance of the surface it contains. This is called <em>light injection</em>.

Answering question 2 by casting thousands of rays per pixel is accurate but far too slow for real time. The voxel cone tracing insight is that the answer to question 2 doesn't need to be exact. Build a mip hierarchy of the radiance grid — coarser levels are pre-blurred averages of finer ones — and then, for each shaded pixel, march a small number of cone-shaped queries through that hierarchy, automatically reading from finer levels when the cone is narrow and from coarser levels when it has widened. Each cone accumulates both the incoming light and the opacity blocking it in one front-to-back pass. Five or six cones cover the hemisphere above a surface and together approximate the entire indirect diffuse contribution.

The result: soft, area-light-quality indirect illumination and colour bleeding that updates every frame, proportional to the cost of a handful of 3D texture lookups rather than thousands of rays.

---

## the pieces of the pipeline

### step 1 — voxelizing the scene

The first task each frame is converting the triangle mesh scene into a 3D grid of voxels. This uses the GPU's standard rasterization hardware: the scene is rendered three times (once from each of the X, Y, and Z axes), and wherever a triangle covers a grid cell the cell is marked and material attributes — albedo, surface normal, and emission — are written into 3D textures via image atomic operations. Crassin and Green's 2012 OpenGL Insights chapter describes how to build a sparse voxel octree (SVO) directly from the GPU rasterizer without an intermediate full grid, using draw-indirect commands and atomic counters [2][local PDF](../papers/crassin-green-2012-octree-sparse-voxelization-gpu.pdf).

For static geometry, voxelization can be cached. For dynamic objects — characters, moving props — only the cells they occupy need to be re-voxelized each frame or when the object moves. This is how the system stays dynamic: the voxel grid tracks the scene.

The [mesh voxelization](../generating/mesh-voxelization.md) page covers the voxelization step in detail, including conservative voxelization and the DDA-based rasterization approach.

### step 2 — light injection

Once the geometry is in the voxel grid, direct lighting must be injected into it. For each occupied voxel the system samples any shadow maps, evaluates the direct lighting equation using the stored normal and albedo, and writes the resulting outgoing radiance back into the voxel. This turns the grid from "geometry description" into "radiance volume" — each cell now represents a small patch of lit surface.

A critical detail here is <em>anisotropic radiance storage</em>. A naive approach stores a single radiance value per voxel, but a thin wall has light on one face and darkness on the other: one scalar value would average them and cause light to bleed through. The solution is to store six separate radiance values per voxel, one for each face direction (±X, ±Y, ±Z). Each cone then selects which of those six faces to sample based on its own direction [3]. This is the primary mechanism that keeps thin-wall light leaking under control — though it does not eliminate it entirely.

### step 3 — the radiance mip pyramid (or clipmap)

The single-resolution radiance grid can only answer queries at the scale of one voxel. Indirect light arrives from surfaces at many distances, so the system needs a hierarchy. A 3D mip map is constructed: level 0 is the original voxel grid, level 1 halves the resolution in each axis (covering 8 original cells per new cell), level 2 halves again, and so on. Each level stores the averaged (and blended) radiance of the level below. Storing six directional values at every level gives the hierarchy the anisotropy needed to support cone tracing from any direction.

The original 2011 paper uses a sparse voxel octree for this hierarchy [1]. NVIDIA's VXGI middleware, which extended the technique for production games, replaced the octree with a <em>clipmap</em> — a set of nested 3D textures at fixed resolution centred on the camera, where each level covers a progressively larger area of the world at progressively lower resolution [4]. Clipmaps are simpler to update incrementally as the camera moves (only the newly uncovered shell at each level needs refreshing) and avoid the pointer-chasing cost of octree traversal. Q-Games' cascaded implementation in *The Tomorrow Children* used six clipmap cascades, each doubling the covered volume, and achieved ~3ms for the cascade updates by updating distant levels less frequently [5].

The [octrees and sparse voxel octrees](../storing/octrees-and-svo.md) page explains the SVO data structure in depth.

### step 4 — cone tracing

Cone tracing is where indirect light is gathered for each screen pixel. At a shaded surface point, the system traces a small number of cones into the hemisphere above the surface normal. Each cone starts narrow and widens as it advances — physically, a widening cone corresponds to querying light from an increasingly large solid angle as distance grows.

The widening is the key that makes the mip hierarchy useful. At a small distance `t` from the surface, the cone's footprint diameter is approximately `2t × tan(aperture/2)`. From that diameter the system computes the appropriate mip level as `log₂(diameter / voxel_size)`. As the cone advances, it automatically steps into coarser mip levels, gathering contributions from wider and wider regions of the scene without any additional branching. Each step accumulates radiance and opacity using front-to-back alpha compositing; marching stops when opacity reaches 1.0 (fully occluded) or the cone exits the grid.

**Diffuse cones** cover the full hemisphere. The original Crassin et al. implementation uses one cone aligned with the surface normal and five more distributed 60° away from it, each with an aperture of about 60° [1][3]. Six cones sample the entire hemisphere without gaps. The result is smooth, low-frequency indirect diffuse light — the hallmark of bounced GI.

**Specular cones** trace a single narrow cone in the reflected view direction. The aperture is tied to material roughness: a mirror-smooth surface traces a cone of ~10° aperture (approaching a single ray), while a rough surface opens it to 30° or more and reads from coarser mip levels, naturally producing blurry reflections. The roughness-to-aperture mapping approximates the GGX lobe that a physical BRDF would require.

The cone marching itself does not use the DDA grid traversal described in [ray traversal with DDA](../rendering/grid-ray-traversal.md) — that traversal steps through individual voxels at a constant grid resolution. Cone tracing instead samples trilinearly (or quadrilinearly, interpolating between mip levels) from 3D textures at continuously varying mip levels. This is significantly cheaper per sample but trades geometric precision for speed.

### step 5 — combining into the final frame

The gathered indirect radiance from the diffuse cones is blended with the directly-lit result in screen space. Specular cone results contribute to the reflection term. Both passes typically run at half or quarter resolution and are upsampled with geometry-aware filters to the final resolution — the spatial blur of the cones already makes high-frequency detail impossible to recover, so the resolution reduction costs little quality.

For two-bounce GI, the scene's voxels can themselves be treated as secondary emitters: inject the cone-traced indirect light from the first pass back into the voxel grid before running the second cone-trace pass. The 2011 paper reports 25–70 FPS for two-bounce diffuse and specular GI at this generation of hardware [1].

---

## why it's real-time and what breaks

### what makes it fast

- **Coarse geometry.** The voxel grid is far less detailed than the triangle mesh. Shading complexity doesn't matter — the grid captures only the light leaving surfaces, not the triangles themselves.
- **Few samples.** Five or six diffuse cones replace thousands of rays. The mip hierarchy lets each sample cover vast distances cheaply.
- **GPU texture hardware.** The trilinear/quadrilinear sampling and mip hierarchy are exactly what GPU texture units do in silicon. Cone tracing is a sequence of ordinary 3D texture fetches.
- **Approximate, not exact.** The technique accepts visible approximation in exchange for orders-of-magnitude speed.

### the main artifacts

- **Light leaking.** Even with anisotropic six-face storage, coarse voxels representing thin walls still allow some leakage. At the mip levels where cones spend most of their travel, a thin wall might occupy only a fraction of a voxel and be averaged away. This is the most persistent quality problem with the technique [6].
- **Temporal lag.** Re-voxelizing the scene and re-injecting direct light takes at least one frame. Fast-moving objects or rapidly changing lights produce a 1–2 frame delay in indirect light, visible as a ghosting or smearing artifact.
- **Resolution limits.** Fine geometric detail — a mesh of cables, a chain-link fence — can vanish at voxel grid resolution, either blocking indirect light incorrectly (over-occluding) or failing to occlude at all. Very large scenes force a coarser grid, reducing quality.
- **Voxelization cost.** Re-voxelizing dynamic objects every frame costs GPU time proportional to the number of moving triangles. Production implementations limit this by re-voxelizing only changed regions or updating less frequently.

---

## contrast with alternatives

The choice between voxel cone tracing and other real-time GI techniques is a direct tradeoff between coverage, quality, and hardware requirements.

| | voxel cone tracing | SDF GI | hardware ray-traced GI |
|---|---|---|---|
| light leaking | common through thin walls | mostly leak-free [6] | essentially none |
| specular quality | approximate (cone width) | good | physically accurate |
| dynamic scenes | full support (re-voxelize) | full support | full support |
| memory cost | high (3D texture × 6 × mip levels) | moderate (SDF per mesh) | low (BVH) |
| hardware requirement | any DX11 GPU | any modern GPU | RT cores required |
| empty-space skip | poor (mip levels help somewhat) | excellent (sphere tracing) | excellent (RT hardware) |
| when shadows fail | leaking through thin walls | missing geometry loses energy | none |

### when to reach for voxel cone tracing

- You need diffuse and rough specular indirect light on any DX11-era GPU, without RT hardware.
- The scene has large, thick geometry (architectural interiors, terrain) where thin-wall leaking is rare.
- You can tolerate 1–2 frame lag on moving objects.
- Budget: roughly 3–10ms for the full pass at 1080p depending on scene complexity and cascade count.

### when to reach for something else

- **SDF GI** (e.g. Godot 4's SDFGI): prefer it when light leaking through thin walls is unacceptable and the scene is mostly static — SDF tracing skips empty space efficiently and misses no geometry [6].
- **Hardware ray-traced GI** (e.g. NVIDIA RTXGI, Lumen on PC): prefer it when RT hardware is available and physical accuracy matters. A 2021 study showed hardware RTGI at ~4ms outperforming software voxel raymarching at ~5ms on equivalent scenes, with superior geometric accuracy [7]. Lumen's engineers found that pure voxel cone tracing suffered from leaking artifacts that no filtering could resolve and moved to distance-field tracing as their primary occlusion structure [8].
- **Precomputed lightmaps with probes** (e.g. [baked ambient occlusion and light](../optimization/baking-ambient-occlusion-and-light.md)): prefer for static scenes where maximum quality matters and runtime cost must be near zero.

The lineage is clear: voxel cone tracing proved the concept of real-time bounced light in 2011 and drove hardware forward. The techniques that follow it — SDF GI, Lumen, RTXGI — are refinements that trade its specific artifacts for their own tradeoffs, not rejections of the core idea.

See also [voxels beyond games](./voxels-beyond-games.md) for uses of voxel radiance grids outside the real-time rendering context.

---

## references

[1] Crassin, C., Neyret, F., Sainz, M., Green, S., and Eisemann, E. (2011). "Interactive Indirect Illumination Using Voxel Cone Tracing." *Computer Graphics Forum*, 30(7), 1921–1930. DOI: 10.1111/j.1467-8659.2011.02063.x. [local PDF](../papers/crassin-2011-voxel-cone-tracing.pdf) · [source](https://research.nvidia.com/publication/2011-09_interactive-indirect-illumination-using-voxel-cone-tracing)

[2] Crassin, C. and Green, S. (2012). "Octree-Based Sparse Voxelization Using the GPU Hardware Rasterizer." In *OpenGL Insights*, CRC Press. [local PDF](../papers/crassin-green-2012-octree-sparse-voxelization-gpu.pdf) · [source](https://research.nvidia.com/labs/rtr/publication/crassin2012voxelization/)

[3] Villegas, J.M. (2016). "Deferred Voxel Shading for Real-Time Global Illumination." *IEEE VIS 2016* / ieeexplore. [source](https://jose-villegas.github.io/post/deferred_voxel_shading/) — detailed implementation reference for anisotropic radiance storage, light injection, and cone parameters.

[4] Panteleev, A. (2015). "NVIDIA VXGI: Dynamic Global Illumination for Games." GTC 2015 talk S5670. [source](https://docs.huihoo.com/gputechconf/gtc2015/S5670-NVIDIA-VXGI-Dynamic-Global-Illumination-for-Games.pdf) — clipmap vs. octree architecture in VXGI.

[5] McLaren, J. (2015). "Graphics Deep Dive: Cascaded Voxel Cone Tracing in *The Tomorrow Children*." *Game Developer*. [source](https://www.gamedeveloper.com/programming/graphics-deep-dive-cascaded-voxel-cone-tracing-in-i-the-tomorrow-children-i-) — six-cascade clipmap, 16 fixed cone directions, 3ms total cost.

[6] Juan Linietsky (Godot Engine) (2020). "Godot 4.0 Gets SDF-Based Real-Time Global Illumination." Godot Engine blog. [source](https://godotengine.org/article/godot-40-gets-sdf-based-real-time-global-illumination/) — SDFGI motivation: VCT light leaking, comparison with SDF approach.

[7] Dobrev, P. et al. (2021). "Real-Time Global Illumination Using OpenGL and Voxel Cone Tracing." *arXiv:2104.00618*. [source](https://arxiv.org/pdf/2104.00618) — performance comparison: VXGI ~5.24ms vs. RTXGI ~3.98ms on equivalent Sponza scene.

[8] Narkowicz, K. (2022). "Journey to Lumen." Personal blog. [source](https://knarkowicz.wordpress.com/2022/08/18/journey-to-lumen/) — Lumen's engineers describe voxel cone tracing attempts and why leaking drove them to distance-field tracing.
