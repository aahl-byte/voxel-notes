<link rel="stylesheet" href="./css/globals.css">

# case studies — voxel engines in practice

The same building blocks — a storage format, a meshing or rendering strategy, an editing model, a simulation layer — can be assembled into a $10 indie heist game, a PlayStation 4 art toy, a universe of 18 quintillion planets, or a research demo that broke the internet. None of those systems is "doing voxels." Each one made a deliberate stack of choices that added up to something very specific.

Reading through five real shipped systems shows something that no single-topic page can: **the choices couple**. The storage format constrains what rendering is possible. The rendering path constrains how big scenes can be. The scene budget constrains what physics is affordable. Every tradeoff sends ripples through every other layer. That's why this page exists as synthesis — to see the whole chain, not just the links.

---

## the coarse model — same parts, different assemblies

Before drilling into any individual system, hold this picture: every voxel engine is built from at most five concerns.

- **Storage** — how the raw voxels are laid out in memory ([dense chunks](../storing/dense-grids-and-chunks.md) vs sparse structures)
- **Surface** — whether you turn voxels into polygons first, or send rays through the grid directly ([meshing](../meshing/blocky-and-greedy-meshing.md) vs [raytracing](../rendering/sparse-voxel-octree-raytracing.md) vs [splatting](../rendering/splatting-and-point-rendering.md))
- **Generation** — where the voxels come from (artist-modeled, procedural density, CSG sculpt)
- **Editing** — whether the world can change at runtime, and how fast
- **Simulation** — physics, lighting, fluid, destruction layered on top

No engine does all of these optimally. Every engine trades some concerns against others. The case studies below name the trade each one actually made.

---

## minecraft — simplicity that scaled to a billion players

### the representation

Minecraft divides the world into vertical columns of 16×16 chunks. Each chunk column is subdivided into 16×16×16 *chunk sections* — 4,096 block cells per section. Rather than storing a full block-state ID per cell (which would take 15 bits × 4,096 entries per section), Minecraft uses a <em>PalettedContainer</em>: a compact local palette that maps small integer indices to block states, so a section with only stone, air, and dirt needs just 2–3 bits per cell instead of 15.

Three palette modes exist, selected automatically by block diversity:

- **Single-valued** — the whole section is one block type; the data array is omitted entirely
- **Indirect** — a local palette (up to ~256 entries); entries packed into 64-bit longs, bits-per-entry sized to the local palette
- **Direct** — all global block-state IDs stored verbatim (15 bits per entry), used only when a section has exceptional variety

Entries pack tightly: at 5 bits per entry, each `long` holds 12 entries (60 bits used, 4 padding). Coordinates index in X→Z→Y order. Biome data uses the same structure at 4×4×4 resolution per section. [1][2]

### meshing and rendering

Minecraft does not raytrace the voxel grid. Instead it generates polygon meshes from chunk data — one mesh per chunk section — and rasterizes them with a conventional GPU pipeline. The technique is <em>greedy meshing</em>: adjacent faces of the same block type and orientation are merged into a single quad, cutting the polygon count dramatically compared to emitting one quad per visible face. The tradeoff is that the merged quads carry a stretched texture rather than a tiled one, which is why the blocky aesthetic is not just stylistic — it is a direct consequence of the meshing strategy. [3]

Lighting runs as a separate pass using <em>flood-fill propagation</em>: two independent channels (sky light and block light) spread through the grid by breadth-first search, each cell receiving the highest neighbor value minus 1. The system approximates global illumination cheaply enough to run on a CPU thread alongside chunk generation. [3]

Chunks outside a configurable view distance are unloaded from memory and serialized to disk. The load/unload boundary defines the world the player sees; terrain beyond it simply doesn't exist until needed.

### the defining tradeoff

Minecraft chose **simplicity and full editability over rendering fidelity**. The palette array is readable, writable, and serializable with trivial code. The greedy-mesh rasterizer runs on any GPU from the last decade. Flood-fill lighting is understandable enough that thousands of modders have extended it. Every technical decision kept the system hackable by a small team (originally one person) and affordable on modest hardware. The result is a game that runs on decade-old phones and has been ported to every platform imaginable. Photorealism was never on the table.

> See [dense grids and chunks](../storing/dense-grids-and-chunks.md) for the palette compression scheme, and [blocky and greedy meshing](../meshing/blocky-and-greedy-meshing.md) for how the face merging works.

---

## teardown — tiny world, every photon earned

### the representation

Teardown (Tuxedo Labs, Dennis Gustafsson, 2020) represents the world as a collection of independent voxel objects — each stored as a 3D volume texture on the GPU, one byte per voxel. Each byte is an index into a 256-entry palette that stores color, roughness, emissivity, reflectivity, and physical material type. This means any individual object supports up to 255 distinct materials at the cost of one byte per cell — extremely memory-efficient. [4][5]

Each object is capped at **256×256×256 voxels**, and each voxel represents a **10 cm cube** in world space, so a maximum object occupies a 25.6 m cube. The world itself is not a single monolithic grid; it is a scene graph of such objects placed in space. A world-space shadow volume texture — for the Marina level it reached **1,752×100×1,500 texels** at 1-bit-per-2×2×2-voxels, representing a 3,504×200×3,000 effective voxel volume — persists across the entire play area and is updated incrementally as destruction happens. [6]

Scenes are authored in MagicaVoxel (.vox files), then loaded and composited into the world. Because Teardown has to fit a full destructible scene into memory at once, levels are necessarily **compact** — Gustafsson has noted a technical limitation on level size in interviews. [7]

### meshing and rendering

Teardown does **not mesh**. The GPU casts rays directly through volume textures using a modified voxel DDA accelerated by mipmapped hierarchical empty-space skipping (effectively a dense octree in texture space). Three trace functions handle different precision-vs-speed tradeoffs: a precise step-by-step traversal for close shadows, a sparse version that adapts to grid density, and a "super-sparse" cone-tracing pass for distant approximations. [6]

The overall pipeline is a deferred renderer with a G-buffer filled by per-object raycasts, followed by stochastic secondary rays (ambient occlusion, soft shadows, specular jitter) resolved by spatiotemporal denoising: spiral blur filters weighted by depth and normal similarity, temporal reprojection (4+ times per frame), and blue-noise sampling for area lights. This produces the characteristic soft, film-like look without a full path tracer. [6]

Global illumination is **not implemented** — the soft lighting is entirely ambient occlusion plus stochastic shadow rays. This is the rendering budget that makes real-time CPU destruction affordable.

### editing and simulation

Destruction is handled on the CPU. When a voxel is removed (by explosion, cutting tool, or vehicle impact), the engine checks for disconnected rigid-body fragments using a connectivity sweep, then spawns those fragments as independent physics objects. This runs on SIMD-parallel CPU threads. For multiplayer, the server sends **destruction commands** — not voxel state — to all clients, which replay them deterministically; new players joining receive a buffer of recent commands. [7][8]

> See [voxel physics and destruction](../simulation/voxel-physics-and-destruction.md) for how disconnected-fragment detection works.

### the defining tradeoff

Teardown chose **small, dense, fully destructible scenes with film-quality soft lighting** over large open worlds. Every voxel that exists has been placed deliberately by a level designer; the world is not procedural. The constraint — scenes small enough to fit in GPU memory — is what makes per-voxel raytracing, real-time destruction, and coherent denoising all viable simultaneously. It is the opposite of No Man's Sky's trade.

---

## dreams — sculpt anything, render without triangles

### the representation

Dreams (Media Molecule, PS4/PS5, 2020) starts from a different premise entirely: players sculpt objects using <em>constructive solid geometry</em> (CSG) — adding and subtracting overlapping shapes, like digital clay. [Signed distance fields](../generating/sdf-and-csg-modeling.md) describe each shape; the scene is an *operationally transformed CSG tree* of SDF primitives that can be composed, unioned, and subtracted in real time. [9]

The CSG tree is not stored as a voxel grid at the authoring level. Instead, at render time, the engine evaluates the CSG tree on-the-fly to produce a **dense multi-resolution point cloud** — a cloud of oriented point splats at the resolution needed for the current view. This evaluation happens on PS4's compute units (no rasterizer involved). [9][10]

### rendering

The rendering pipeline, described by Alex Evans at SIGGRAPH 2015 in "Learning from Failure," is: CSG edit list → compute shader → signed distance fields → pre-filtered and clustered point clouds → software renderer writing directly to a Z-buffer and G-buffer. **No triangles are emitted at any stage.** [10]

The output is a [point splat](../rendering/splatting-and-point-rendering.md) renderer: each point becomes a 2D disk oriented by its surface normal, sized to cover the pixel footprint at its depth. Points at different LODs produce splats at different sizes — coarser geometry produces larger splats, which gives Dreams its characteristic soft painterly look. That look is not a post-process filter; it is the natural result of variable-resolution point sampling. [9][10]

Lighting runs through the same compute-based pipeline. Because there are no triangle meshes, there is no shadow map in the conventional sense; illumination is derived per-splat from the SDF.

### the defining tradeoff

Dreams chose **unlimited sculpting freedom and a wholly unconventional render path** over polygon compatibility and conventional tooling. The system can represent geometry that would be topologically impossible or extremely difficult in polygons (fine filigree, organic blends, negative-space cutouts) because the CSG tree describes the *volume*, not the surface. The cost: the renderer is entirely custom, the asset pipeline is proprietary, and it ran on PS4's compute hardware rather than the vertex/pixel shader path every other PS4 title used. Porting requires re-engineering the entire render stack.

> Evans documented four years and three abandoned renderer prototypes before arriving at this solution — the talk "Learning from Failure" [10] is worth reading as a case study in technical iteration.

---

## no man's sky — planet scale from a density function

### the representation

No Man's Sky (Hello Games, 2016–present) generates terrain from **procedural density functions** — mathematical expressions evaluated at every point in a planet's voxel grid to produce a signed occupancy value. The density function is not stored; it is recomputed on demand from a deterministic seed. This is what makes 18 quintillion unique planets possible: the voxel data for any location on any planet can be regenerated in microseconds from a compact function, without ever persisting a grid to disk. [11][12]

There are no pre-authored voxel assets for terrain (unlike Teardown's .vox files). Procedural generation replaces the artist's hand entirely for landscape — a fundamentally different creative pipeline. [See procedural terrain](../generating/procedural-terrain.md) for how density functions are structured.

### meshing

The density field is then polygonized into a triangle mesh for rendering. Early versions used a marching-cubes variant; subsequent major updates (notably the 2024 Worlds Part I update) adopted <em>dual marching cubes</em>, which produces meshes with lower vertex counts, faster generation, and better memory characteristics than standard marching cubes. [12]

> See [surface nets and dual contouring](../meshing/surface-nets-and-dual-contouring.md) for how dual contouring and dual marching cubes differ from the standard algorithm.

LOD is handled by generating the same density function at progressively coarser voxel resolutions for chunks at greater distances, then blending seams between LOD levels. At planet scale this means dozens of LOD tiers, with the coarsest representing entire mountain ranges as a handful of triangles. [See LOD in engines](./lod-in-engines.md) for the mechanics.

### editing

Runtime terrain editing (digging, terrain deformation with the multi-tool) works by locally modifying the density function — clamping or overriding values in a small radius — and regenerating the affected mesh chunks. Because the mesh is derived from the density, the edit is automatically consistent at all LOD levels that cover that region. [11]

### the defining tradeoff

No Man's Sky chose **infinite procedural scale at the cost of artistic control and edit precision**. The density-function approach makes arbitrary planet generation tractable but means that the terrain is whatever the math produces — fine details require finely tuned functions, not per-voxel painting. And because the mesh is generated per-chunk from a function, the world has no persistent voxel state for terrain between sessions beyond the player's edits (which are stored as delta overrides on the base function). This is the opposite of Teardown's fully stored, fully persistent, fully destructible voxel world.

---

## svo raytracing engines — extreme detail, real limits

Several engines push voxel raytracing toward extreme geometric resolution by storing geometry as <em>sparse voxel octrees</em> (SVOs) or <em>sparse voxel DAGs</em> — structures that represent only the occupied cells of a massive grid. See [sparse voxel octree raytracing](../rendering/sparse-voxel-octree-raytracing.md) for how traversal works.

### euclideon — the unlimited detail controversy

Euclideon (Brisbane, Australia) released a video in 2011 claiming "Unlimited Detail" — a voxel engine that could render more geometry than any polygon engine. The video showed a forested island with photorealistic foliage at high framerates, with claims that polygon-based rendering was fundamentally obsolete.

The technical reality, dissected in detail by the graphics community, was more modest. Euclideon's system is a <em>point-cloud search engine</em>: it indexes a large set of 3D points, then at render time performs a screen-space search to find which points project onto each pixel, displaying exactly one point per pixel. For a 1024×768 display, that means finding and drawing 786,432 visible points per frame. [13]

This approach — projecting a pre-built point cloud to fill screen pixels — traces back to techniques described in papers from the 1980s and is not categorically different from established sparse-voxel rendering. The specific limitations the community identified:

- **No dynamic lighting** in the original demos — illumination was prebaked from offline renderers. Adding per-frame dynamic lights to a point-cloud renderer of this design requires re-lighting every visible point per frame. [13]
- **No skeletal animation** — animating a point cloud with complex deformers (joints, skinning) requires per-frame point repositioning at massive scale, which was not demonstrated. [13]
- **Static scenes only** — the showcased content was scanned real-world terrain converted to point clouds, not dynamically generated or editable geometry. [13]

Euclideon later found a legitimate niche: **geospatial LiDAR visualization** for architecture, mining, and infrastructure, under the product name Geoverse. Rendering a 3D scan of a city block or a mine shaft at human-scale detail is exactly what the point-cloud approach does well. It never shipped as a game engine. The company entered administration; the gaming promise did not materialize. [14]

### atomontage — voxel simulation at scale

Atomontage (Branislav Siles, 2019–present) represents a more technically grounded attempt at real-time voxel rendering with dynamic content. Siles, who has spent 15+ years on volumetric graphics research, built a system targeting **billions of fully simulated voxels** with real-time editing, destruction, and physics — not just static point clouds. [15]

Unlike Euclideon's prebaked content, Atomontage targets dynamic worlds: voxels that can be added, removed, and physically simulated at runtime. The engine uses sparse representations compatible with ray tracing hardware, with streaming to handle scenes too large to fit in GPU memory. As of 2024 the engine remains in development and limited public demonstrations have been shown; it has not shipped in a consumer title.

### john lin's engine — research-grade raytracing

John Lin, an independent developer, has demonstrated over several years a real-time voxel engine built on Vulkan ray tracing extensions that achieves visually rich results: ray-traced soft shadows, ambient occlusion, and specular highlights at interactive frame rates over a sparse voxel world. The engine has gone through multiple revisions, with each iteration shrinking the voxel size and increasing detail. [16]

Lin's own analysis of the "perfect voxel engine" is a useful reality check: he notes that SVOs excel at rendering and storage but are awkward for collision detection, pathfinding, and dynamic attribute updates — the things a game actually needs beyond the camera. The lesson is that a raytracing showcase and a game engine are different systems. [16]

---

## cross-cutting patterns

Reading across all five systems, the same structural patterns appear.

### the storage-render coupling is tight

Every engine's rendering approach is locked to its storage format. Teardown can raytrace in real time because its scenes are small enough to fit in GPU memory as volume textures. No Man's Sky can have planet-scale terrain because it never stores the terrain — it recomputes it. Dreams can use point splats because its CSG tree is the canonical form and polygons are never generated. Minecraft can run on any GPU because its palette-array chunks feed a conventional rasterizer. You cannot mix these freely: switching Teardown to a procedural terrain would require rethinking the entire shadow map and destruction model.

### editability and scale trade directly

The more of the world that is fully editable at voxel resolution, the smaller the world must be (or the less of it can live in memory at once). Teardown: fully editable, compact scenes. Minecraft: fully editable, but the voxel is large (1 m³) and chunk loading hides the scale. No Man's Sky: editable at a limited radius, planet scale, but most of the planet is never persisted. Dreams: editable CSG, but scenes are small (a room, a character) not open worlds.

### rendering fidelity costs scene size

Teardown's stochastic soft shadows and temporal denoising require per-frame ray budgets that only work over scenes measured in hundreds of meters. No Man's Sky's planet-scale needs conventional rasterization with a polygon mesh — you cannot raytrace a planet-sized scene in real time on consumer hardware today. Euclideon's point-cloud renderer achieves very high geometric density but pays for it with static content and no dynamic lighting.

### simplicity compounds

Minecraft's palette arrays, greedy meshing, and flood-fill lighting are each simple in isolation. Combined, they produce a system that runs on any hardware, can be modded by millions of people, and has been maintained and extended by a large team for 15 years. Complexity in any one layer would have propagated into every other. Dreams' custom compute renderer is the counterexample: maximum expressiveness, but at the cost of a system that only its original authors can maintain and that shipped on exactly one hardware architecture.

---

## what to copy for your project

The case studies above point back to a set of decision pages. The choice you make on each one propagates through your whole stack.

| you want | look at | tradeoff to accept |
|---|---|---|
| infinite or very large worlds | No Man's Sky — procedural density + chunked meshing | no per-voxel state by default; edits are deltas |
| full destruction at voxel resolution | Teardown — per-object volume textures + CPU physics | scenes must fit in GPU memory; small worlds |
| smooth organic shapes, arbitrary CSG | Dreams — SDF/CSG tree + point splatting | no conventional rendering pipeline; complex toolchain |
| simple, portable, moddable | Minecraft — palette chunks + greedy mesh + rasterizer | coarse voxels; no interior lighting without bake |
| extreme geometric detail (research/viz) | SVO/DAG raytracing (Lin, Euclideon) | static or low-dynamic content; large memory |

The specific decision pages:
- How to store your voxels: [dense grids and chunks](../storing/dense-grids-and-chunks.md) and [the storage problem](../storing/the-storage-problem.md)
- Whether to mesh or raytrace: [why mesh voxels](../meshing/why-mesh-voxels.md) and [choosing a render path](../rendering/choosing-a-render-path.md)
- Smooth-surface meshing: [surface nets and dual contouring](../meshing/surface-nets-and-dual-contouring.md)
- SDF-based sculpting: [SDF and CSG modeling](../generating/sdf-and-csg-modeling.md)
- LOD at scale: [LOD in engines](./lod-in-engines.md)
- Physics and destruction: [voxel physics and destruction](../simulation/voxel-physics-and-destruction.md)
- The full engine architecture: [anatomy of a voxel engine](./anatomy-of-a-voxel-engine.md)

---

## references

[1] Minecraft Wiki contributors. "Java Edition protocol/Chunk format." *Minecraft Wiki*. [source](https://minecraft.wiki/w/Java_Edition_protocol/Chunk_format)

[2] Fabric MC. "PalettedContainer (yarn 1.20.1+build.1 API)." *FabricMC Maven Docs*. [source](https://maven.fabricmc.net/docs/yarn-1.20.1+build.1/net/minecraft/world/chunk/PalettedContainer.html)

[3] Svendsen, Mikola. "Meshing in a Minecraft Game." *0fps.net*, June 2012. [source](https://0fps.net/2012/06/30/meshing-in-a-minecraft-game/) (Canonical reference on greedy meshing and flood-fill lighting for Minecraft-style engines.)

[4] Gustafsson, Dennis. "The Spraycan." *Voxagon Blog*, December 3, 2020. [source](https://blog.voxagon.se/) (Details the 8-bit palette material system: color, roughness, emissivity, reflectivity, physical type.)

[5] Teardown Wiki contributors. "Voxels." *Teardown Wiki*. [source](https://teardown.fandom.com/wiki/Voxels) (Confirms 10 cm per voxel; 256³ max object size; one byte per voxel palette index.)

[6] Montoya, Juan Diego. "Teardown Frame Teardown." *acko.net*, 2021. [source](https://acko.net/blog/teardown-frame-teardown/) (Full frame dissection: G-buffer layout, shadow volume texture dimensions, DDA traversal, denoising pipeline.)

[7] Gustafsson, Dennis. "Teardown dev's top priority is always physics." *TechRadar*, 2023. [source](https://www.techradar.com/gaming/consoles-pc/teardown-devs-top-priority-is-always-physics-the-game-can-come-later)

[8] "Teardown Developer Breaks Down Multiplayer and Voxel Destruction Tech." *80.lv*, 2023. [source](https://80.lv/articles/teardown-developer-breaks-down-multiplayer-and-voxel-destruction-tech) (Multiplayer deterministic command-replay architecture; SIMD CPU destruction.)

[9] Evans, Alex. "Learning from Failure: a Survey of Promising, Unconventional and Mostly Abandoned Renderers for 'Dreams PS4'." *Advances in Real-Time Rendering, SIGGRAPH 2015*. [source](https://www.mediamolecule.com/blog/article/siggraph_2015) · [slides PDF](http://media.lolrus.mediamolecule.com/AlexEvans_SIGGRAPH-2015.pdf)

[10] SIGGRAPH 2015 Advances in Real-Time Rendering course page. [source](https://advances.realtimerendering.com/s2015/) (Dreams pipeline: CSG edit list → compute → SDF → multi-resolution point cloud → software renderer; no triangles.)

[11] Murray, Sean. "Continuous World Generation in 'No Man's Sky'." *GDC Vault*, 2018. [source](https://www.gdcvault.com/play/1024265/Continuous-World-Generation-in-No) (Pipeline: density function → polygonization via marching cubes → texturing → population/simulation.)

[12] Hello Games. "Worlds Part I Update." *No Man's Sky*, July 2024. [source](https://www.nomanssky.com/worlds-part-I-update/) (Confirms adoption of dual marching cubes for reduced vertex count, faster generation, lower memory.)

[13] Elias, Nick. "Euclideon Explained." *CoderNotes*, 2011. (Technical debunking: point-cloud search algorithm, prebaked lighting, no skeletal animation, static content limitation. URL: codersnotes.com/notes/euclideon-explained — SSL certificate error prevents direct fetch; content confirmed via secondary sources [14].)

[14] "Euclideon." *Wikipedia*. [source](https://en.wikipedia.org/wiki/Euclideon) (Corporate history, Geoverse pivot, administration.)

[15] Siles, Branislav. "Atomontage reveals voxel 3D simulation technology for games." *VentureBeat / GamesBeat*, 2019. [source](https://gamesbeat.com/atomontage-reveals-voxel-3d-simulation-technology-for-games/) (15 years R&D, targeting billions of dynamic simulated voxels, runtime editing and physics.)

[16] Lin, John. "The Perfect Voxel Engine." *Voxely.net Blog*. [source](https://voxely.net/blog/the-perfect-voxel-engine/) (Critique of SVO limitations for games: collision, pathfinding, dynamic attributes; multi-format approach.)
