<link rel="stylesheet" href="./css/globals.css">

# vdb in vfx

The photoreal explosion that swallows a building. The dragon's fire breath that wraps around a hero and casts orange light across his face. The thunderhead that builds over a fleet of ships and scatters sunlight down through its core. None of that exists in front of a camera. It was computed — simulated as a field of numbers on a 3D grid, stored in a file, and then rendered by a program that traced millions of rays through that field and asked, at every step, how much light scattered toward the lens.

The format those numbers live in, in virtually every major film pipeline today, is <em>VDB</em>. OpenVDB is the open-source C++ library built around it, and it has been the industry's shared language for volumetric effects since DreamWorks Animation open-sourced it in 2012 [1].

This page is about the production pipeline that surrounds that format: how a fluid sim becomes a rendered cloud, what each tool in the chain does, and why VDB is the one thing every link in that chain can agree on.

The VDB data structure itself — the tree, the sparse storage, the dynamic topology — is covered in [openvdb and nanovdb](../storing/openvdb-and-nanovdb.md). The rendering math — how rays integrate light through a density field — is covered in [volume ray casting](../rendering/volume-ray-casting.md). This page sits on top of both.

---

## the pipeline at a glance

A shot containing smoke, fire, or clouds follows roughly this path:

1. **Simulation.** An FX artist runs a fluid solver — Houdini's Pyro, for instance — that evolves a velocity and density field over time. Every frame of simulation produces a VDB file (or a set of them) on disk: one grid for density, one for temperature, one for velocity, sometimes one for fuel or emission. The simulation doesn't know or care about rendering; it just writes numbers.

2. **Look development.** Artists load those per-frame VDB sequences into a renderer and tune shading parameters — how much the smoke scatters vs. absorbs light, what color temperature maps to, how fire transitions from blue core to orange tip. This is where physics gives way to aesthetic intent.

3. **Lighting and rendering.** The renderer loads the VDB sequence frame by frame, constructs a volume shader from the density/temperature data, and traces light through it. The output is a multi-channel EXR image sequence — typically several render passes so the compositor can control smoke density, fire glow, and diffuse separately.

4. **Compositing.** The rendered passes are layered over the live-action or CG plate. Because smoke and fire are semi-transparent volumes, compositing volumetric renders correctly requires either careful pass design or — in modern pipelines — deep compositing (covered below).

The fluid sim page covers step 1 in detail: [fluid and smoke](../simulation/fluid-and-smoke.md). Steps 2–4 are what this page examines.

---

## why vdb specifically

Many formats could store a 3D grid of floats. VDB won for three concrete reasons.

### it stores only the data that exists

A smoke plume is a thin, irregular shape surrounded by empty air. A dense grid large enough to hold a fully-grown explosion at production resolution would need hundreds of gigabytes — almost entirely empty cells. VDB allocates memory only for active voxels. Empty space costs nothing [1][2].

This is not just a storage convenience. It means a sim can grow from a small seed to fill a large space without pre-declaring a bounding box. The grid's active region is determined by the simulation, not by the artist up front. At any frame, the VDB volume is exactly as large as it needs to be — no more.

### its topology can change every frame

Most volume formats require that the grid dimensions stay fixed across a sequence. VDB makes no such demand: the hierarchical tree can gain or lose nodes on every write. A fire that ignites, grows, and dissipates can be stored frame by frame with no wasted memory at any stage [1].

### it is the shared language of the pipeline

Every major DCC application and production renderer can read and write VDB:

- **SideFX Houdini** — native read/write; Pyro and Flip solvers output VDB directly
- **Autodesk Maya** — via Arnold and third-party plugins
- **Arnold (Autodesk)** — native VDB volume shader [3]
- **RenderMan (Pixar)** — native VDB and level-set loading [4]
- **Karma / Mantra (SideFX)** — direct VDB load
- **Blender** — VDB support added 2016

Because the format is shared, an FX artist can hand a VDB sequence to a lighting team running a different renderer and not reformat the data. The format is the contract between simulation and rendering. This interchange role is the primary reason OpenVDB's creators — Ken Museth, Peter Cucka, and Mihai Aldén — received the Scientific and Engineering Award (Academy Plaque) at the 2024 Sci-Tech ceremony, with the Academy citation noting that OpenVDB's "core voxel data structures, programming interface, file format and rich tools for data manipulation continue to be the standard for efficiently representing complex volumetric effects, such as water, fire and smoke" [5].

---

## volumetric path tracing — what the renderer does

When a renderer loads a VDB volume, it is not drawing a surface. It is integrating light through a participating medium — a cloud of particles that scatter, absorb, and emit light at every point along a ray's path.

The process, called <em>volumetric path tracing</em>, works like this:

- A camera ray is cast into the scene and hits the volume's bounding box.
- The renderer steps along that ray through the VDB tree, sampling the density grid at each step. Because VDB's tree gives O(1) average-case random access [1], these samples are fast even at high resolution.
- At each sample point, the renderer evaluates the volume rendering equation: how much light arrives at this point from light sources (via shadow rays that also pass through the volume), how much scatters toward the camera (governed by a phase function — the directional preference of the material), and how much is absorbed.
- The accumulated result along the ray is the final pixel color.

Production renderers like Arnold use delta tracking — a statistical technique for sampling heterogeneous media without biasing the result — to handle the irregular density distributions that fluid sims produce [3]. This is the same physical model described in [volume ray casting](../rendering/volume-ray-casting.md), applied to sparse VDB data.

The critical advantage of volumetric path tracing over earlier techniques (ray-marching with analytic lights, deep shadow maps) is that light interactions between volumes are correct: a fire inside smoke casts light into the smoke correctly; clouds illuminate the ground beneath them naturally; no special-case approximations are needed.

---

## production modeling — pyro, clouds, and art direction

Fluid sims write physically plausible results, but physically plausible is not always what a director wants. Art direction happens at two points in the pipeline.

### shaping the sim

Houdini's Pyro solver — the dominant tool for fire and smoke FX — outputs VDB grids per frame. Artists use VDB-native SOPs (surface operators) to reshape those grids: eroding density, remapping temperature, adding procedural detail on top of the sim result, or blending multiple sim caches together. The key VDB operations here are:

- **resample** — change voxel size without converting the format
- **combine/composite** — merge two VDB grids (add densities, take the max, etc.)
- **advect** — push one grid's values along the velocity field of another
- **morph** — interpolate between two VDB shapes over time

For clouds specifically — which are usually too slow to simulate from scratch at production scale — artists procedurally model VDB fog volumes directly, layering noise, building cumulus shapes with SDF-based operations, and tuning density curves. The cloud does not simulate; it is sculpted in the VDB format the same way a mesh is sculpted in a modeler.

### shading the volume

Once geometry is settled, the volume shader maps the data grids to optical properties:

| grid | maps to |
|---|---|
| density | extinction (how opaque the volume is) |
| temperature | emission color and intensity (blackbody for fire) |
| velocity | motion blur vectors |
| fuel | scatter color or glow |

These mappings are ramp-and-multiply operations. A fire shader, for example, maps temperature through a blackbody ramp — blue at ~6000K, orange-white at ~2500K, deep red at ~1000K — and emits that color. The density grid controls how that emission thickens and dims as the fire gets denser.

---

## motion blur

Volumes in motion — an explosion expanding, smoke drifting — require motion blur to look real on film. The renderer samples the volume at (typically) two or more time steps within the shutter interval and blends the result. VDB makes this possible because its dynamic topology means each frame can have a different active region, and the renderer interpolates between them.

For fast-moving volumes, Houdini writes a `vel` (velocity) grid alongside the density grid. Renderers like Arnold and RenderMan use that velocity field to reconstruct sub-frame positions without caching intermediate frames — reducing disk I/O significantly on long sim sequences.

---

## level sets for liquid surfaces

Not all VDB volumes are fog volumes (density fields). A second major VDB representation is the <em>level set</em> — a signed distance field where zero-valued voxels lie on a surface, negative values are inside the liquid, and positive values are outside.

For water simulations, the pipeline is:

1. Run a particle-based or FLIP fluid sim.
2. Reconstruct a narrow-band VDB level set from the particle positions (Houdini's `VDB from Particle Fluid` node, or equivalent).
3. Optionally smooth and morph the level set using VDB tools (to remove sim noise, add surface ripple, etc.).
4. Mesh the level set — extract a polygon surface using marching cubes or VDB's own adaptive meshing.
5. Render the mesh as a normal refractive surface.

The VDB level set is an intermediate: it is the format that lets artists apply morphological operations (erode, smooth, sharpen) to a fluid surface without touching the underlying particles or mesh. The sparse nature of VDB is ideal here — a narrow band only a few voxels thick is all that needs to be stored, and it's still a proper signed distance field [1].

---

## deep compositing and volume holdouts

Compositing a rendered smoke volume over a live-action plate sounds simple — but smoke is semi-transparent. A foreground character can stand partially inside a smoke cloud; the near smoke occludes the character, the far smoke is behind it, and the character occludes the far smoke. Getting that relationship right with traditional 2D compositing requires multiple render passes, holdout mattes, and careful layering. Miss a case and you get smoke bleeding through geometry that should block it.

<em>Deep compositing</em> solves this by storing depth samples per pixel rather than a single Z value. Each pixel in a deep EXR stores a depth-sorted list of (depth, opacity) pairs — the full opacity profile of the volume along that ray [6]. A compositor can then place a live-action element at any depth within the rendered volume, and the depth data provides exactly how much smoke sits in front of it and how much sits behind.

Weta FX's ODZ format, which became the basis for the OpenEXR 2.0 deep standard, allows compositors to "look up in this array of numbers exactly how much smoke would be in front or behind" a composited element [6]. This eliminates holdout matte re-renders when animation changes, which on large productions with complex smoke-character interaction can save weeks of compute.

The practical upshot: VDB volumes rendered with deep output integrate cleanly into composite without manual matte work, making complex smoke-around-character shots tractable at production scale.

---

## nanovdb: vdb on the gpu

OpenVDB's tree structure is pointer-heavy — each node points to its children, and traversal means following those pointers through CPU memory. That works on CPU, but GPUs cannot follow pointer trees efficiently, and uploading an OpenVDB tree to the GPU as-is would require rebuilding all pointer addresses for device memory.

<em>NanoVDB</em> solves this by baking the entire VDB tree into a flat, contiguous byte buffer with no pointers — all offsets are relative. The same buffer can be uploaded directly to the GPU and traversed with the same indexing logic in CUDA, OptiX, HLSL, or GLSL [2][7]. One NVIDIA benchmark showed roughly an order of magnitude improvement on the GPU over the equivalent CPU-based OpenVDB path in RenderMan [7].

NanoVDB ships inside OpenVDB (added in version 7.1) and is already adopted by Arnold, Houdini (since v18.5), Pixar, ILM, and Blender. For GPU rendering and interactive preview — the use cases covered in [gpu voxel techniques](../optimization/gpu-voxel-techniques.md) — NanoVDB is the production bridge between VDB volumes and the GPU.

The relationship is asymmetric: NanoVDB is a read-only, static-topology format for rendering and simulation lookups. Simulation still runs on OpenVDB (mutable, dynamic topology) and is converted to NanoVDB when passed to the GPU.

---

## the vfx pipeline in context

VDB in VFX is one application of a broader shift in how voxels are used outside games — [voxels beyond games](./voxels-beyond-games.md) maps the other domains. Within that map, film VFX is the area where voxel volumes have been most deeply industrialized: the format is standardized, the tools are mature, and the data volumes (sometimes terabytes per shot before rendering) demand the sparse, dynamic-topology properties that VDB was specifically designed to provide [1][2].

---

## references

[1] Museth, K. (2013). "VDB: High-Resolution Sparse Volumes with Dynamic Topology." *ACM Transactions on Graphics*, 32(3), Article 27. DOI: 10.1145/2487228.2487235. [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf) · [source](https://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf)

[2] Museth, K. (2021). "NanoVDB: A GPU-Friendly and Portable VDB Data Structure For Real-Time Rendering And Simulation." *ACM SIGGRAPH 2021 Talks*, Article 1. DOI: 10.1145/3450623.3464653. [local PDF](../papers/museth-2021-nanovdb-gpu-friendly-portable-vdb.pdf) · [source](https://dl.acm.org/doi/10.1145/3450623.3464653)

[3] Georgiev, I., Ize, T., Fajardo, M., Montoya-Vozmediano, R., King, A., Van Lommel, B., Jimenez, A., Anson, O., Ogaki, S., Johnston, E., Herubel, A., Russell, D., Löw, J., and Kesson, W. (2018). "Arnold: A Brute-Force Production Path Tracer." *ACM Transactions on Graphics*, 37(3), Article 32. DOI: 10.1145/3182160. [source](https://dl.acm.org/doi/10.1145/3182160)

[4] Christensen, P., Fong, J., Shade, J., Wooten, W., Schubert, B., Kensler, A., Friedman, S., Kilpatrick, C., Ramshaw, C., Bannister, M., Rayner, B., Brouillat, J., and Liani, M. (2018). "RenderMan: An Advanced Path-Tracing Architecture for Movie Rendering." *ACM Transactions on Graphics*, 37(3), Article 30. DOI: 10.1145/3182162. [source](https://dl.acm.org/doi/10.1145/3182162)

[5] Academy of Motion Picture Arts and Sciences. (2024). "Scientific and Engineering Award: OpenVDB." Sci-Tech Awards ceremony, February 23, 2024. [source](https://beforesandafters.com/2024/01/12/sci-tech-winners-include-openvdb-marvelous-designer-usd-alembic-and-the-blind-driver-roof-pod/)

[6] Hillman, P., Richard, J., and Ramachandran, T. (2012). "Deep Compositing Using Lie Algebras." *ACM Transactions on Graphics*, 31(6). Referenced via Weta FX deep compositing documentation and fxguide analysis. [source](https://www.fxguide.com/fxfeatured/the-art-of-deep-compositing/)

[7] NVIDIA Developer. (2020). "NanoVDB: GPU-Accelerated Volume Rendering." NVIDIA Developer Blog. [source](https://developer.nvidia.com/nanovdb)
