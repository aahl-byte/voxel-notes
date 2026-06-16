<link rel="stylesheet" href="./css/globals.css">

# ways to render voxels

You have a grid of voxels. You want pixels on screen. That gap — from 3D data to 2D image — is what a render path closes.

There are three fundamentally different ways to close it, and the right choice depends on two things: whether you need to see *through* the volume or only its surface, and whether your scene changes fast enough that rebuilding a mesh would be too slow.

Pick the wrong path and you pay in performance, quality, or both. Pick the right one and the rest of the rendering work follows naturally from the tools your pipeline already has.

The full picture of how voxel data flows from storage to screen lives in [the voxel pipeline](../foundations/the-voxel-pipeline.md).

---

## the three paths at a glance

A coarse model first — enough to reason about the choice before diving into mechanics:

| path | what you actually do | opaque surface or see-through volume? | dynamic scenes? |
|---|---|---|---|
| **mesh rasterization** | extract a triangle mesh, then render it normally | surface only | slow to update |
| **direct ray marching** | cast rays straight through the grid — no mesh | surface (opaque hit) | yes, trivially |
| **volume rendering** | accumulate light all the way through the grid | see-through volume | yes |

One axis separates mesh rasterization from the other two: do you convert voxels into triangles first, or do you send rays directly through the data? The second axis separates direct ray marching from volume rendering: do you stop at the first solid voxel, or do you keep going and blend everything together?

---

## path 1 — extract a mesh, then rasterize it

The voxels are not what the GPU renders. Instead, you run an algorithm — most often marching cubes — that reads the voxel grid and emits a triangle mesh describing the surface. From that point on, the GPU pipeline treats it exactly like any other mesh: vertices through a vertex shader, triangles rasterized, pixels shaded. See [why mesh voxels](../meshing/why-mesh-voxels.md) for a full treatment of the meshing step.

This is <em>mesh rasterization</em>: render a surface mesh that was extracted from the voxel data rather than the voxels themselves.

**Why reach for it:**

- Every existing game engine tool — LOD systems, shadows, skinning, PBR materials, post-processing — works without modification.
- Triangle rasterization on modern GPUs is the most heavily optimized path in graphics hardware.
- Output quality is high: the mesh is a smooth approximation of the voxel surface, not blocky cubes.

**The cost:**

- The mesh must be extracted (or re-extracted) any time the voxels change. For static terrain or pre-baked geometry this is fine. For terrain you carve in real time, re-meshing must happen fast enough to stay ahead of the player — often in background threads.
- You have *lost* the voxel data by the time the renderer sees it. The mesh carries no interior information: you cannot look inside it, and a bullet cannot query density mid-flight.

**When to use:**

- Terrain that is mostly static, or changes only in local patches.
- Scenes where you need full compatibility with an existing rendering pipeline.
- Cases where the smooth surface quality of marching cubes or dual contouring is worth the extraction overhead.

---

## path 2 — cast rays directly through the grid

Instead of converting voxels to triangles, you cast a ray for each pixel and step through the voxel grid until the ray hits a solid voxel. You read the voxel's color or material, shade it, and write the pixel. No mesh, ever.

The stepping algorithm on a uniform grid is called <em>ray marching</em> (specifically the DDA algorithm — Digital Differential Analyser — which advances the ray to each successive grid cell boundary in O(1) per step). On a sparse tree the algorithm descends the hierarchy to skip large empty regions in one jump rather than stepping cell by cell, which is what makes [sparse voxel octree raytracing](./sparse-voxel-octree-raytracing.md) practical at high resolution. The basic uniform-grid version is covered in [grid ray traversal](./grid-ray-traversal.md).

The foundational paper here is Laine & Karras (2010), which showed that sparse octree ray casting is competitive with triangle rasterization at much higher geometric detail [1, local PDF](../papers/laine-karras-2010-efficient-sparse-voxel-octrees.pdf).

**Why reach for it:**

- Dynamic content is free: if a voxel changes, the next frame sees the new value with no re-meshing.
- Geometric detail scales far beyond what any triangle mesh could store at equivalent memory cost — a sparse voxel octree can represent billions of voxels.
- Interior data is available: you can read any voxel at any point along the ray, which enables ambient occlusion, sub-surface queries, or physics lookups without a separate data structure.

**The cost:**

- Per-pixel ray traversal is more expensive than rasterizing a triangle mesh, especially for rays that travel through a lot of empty space before hitting anything.
- Integration with a conventional engine's shadow maps, screen-space effects, and post-processing requires extra care.

**When to use:**

- Scenes with very high voxel density (game worlds at centimeter or millimeter resolution).
- Dynamic destruction or editing where re-meshing every frame is too slow.
- Pure voxel renderers that own the full pipeline.

---

## path 3 — accumulate light through the whole volume

The two paths above stop as soon as they find a solid voxel. Volume rendering does something different: the ray keeps going through *everything*, and at each step it samples density and color, adding a small contribution to the pixel and subtracting a little from the light remaining — until the ray exits the volume or the remaining light reaches zero.

The physical model is <em>volume rendering</em>: each voxel both emits light (contributes color) and absorbs light (blocks what comes from behind). The final pixel value is the integral of those contributions along the ray. A transfer function — a mapping from raw voxel value to color and opacity — controls what the volume looks like: you can make bone opaque and white while tissue stays translucent orange, all from the same CT data, just by adjusting the transfer function.

See [volume ray casting](./volume-ray-casting.md) for the mechanics.

**Why reach for it:**

- The only way to correctly visualize media that is translucent or has a meaningful interior — smoke, fire, clouds, CT/MRI scans, fluid simulations.
- Transfer functions let a single dataset show very different content: the same voxel grid can be a skin-and-bone medical reconstruction or a glowing X-ray view, depending on the function applied.
- Natural output for any data acquired by volumetric sampling (CT, MRI, seismic, point clouds voxelized with density).

**The cost:**

- Every step along the ray is a sample, not a stop — so the cost per pixel is proportional to the depth of the volume, not just the surface.
- Real-time volume rendering at interactive frame rates requires careful optimization: early ray termination when opacity is saturated, empty-space skipping, and often reduced resolution with upscaling.

**When to use:**

- Medical and scientific visualization where interior structure must be visible.
- VFX smoke, fire, clouds, and atmospheric effects.
- Any scene where the interesting content *is* the interior, not the surface.

---

## a second axis — which direction does the algorithm scan?

The three paths above differ in *what* they render. There is a second, orthogonal question: does the algorithm scan pixel-by-pixel outward from the camera, or voxel-by-voxel projecting each one forward?

**Image-order** (per pixel): cast a ray per pixel and gather contributions from the scene. Ray marching and volume ray casting both work this way. Easy to implement on a GPU — one thread per pixel — and supports early termination.

**Object-order** (per voxel): take each voxel, project it into screen space, and splat its contribution onto the image. This is <em>splatting</em> and is covered in [splatting and point rendering](./splatting-and-point-rendering.md). Splatting is object-order volume rendering: instead of reading the volume for each pixel, you push each voxel's footprint onto the image.

Splatting has a natural advantage when voxels are sparse: you only touch the non-empty ones. It has a corresponding disadvantage: you must process them in back-to-front depth order to composite correctly, and that sort is expensive when the camera moves.

Mesh rasterization is also object-order (triangles are projected onto the screen), which is part of why it fits so naturally into existing pipelines.

---

## choosing a path

The full decision guide lives at [choosing a render path](./choosing-a-render-path.md). The short version:

| if you need… | reach for… |
|---|---|
| max compatibility with existing engine | mesh rasterization |
| dynamic voxels, no re-mesh overhead | direct ray marching |
| see-through media (smoke, CT scans) | volume rendering |
| sparse datasets, few non-empty voxels | splatting |
| billions of voxels, huge detail | sparse voxel octree ray tracing |

The render path is one decision in the larger pipeline. The data structure you store voxels in shapes which paths are cheap: a flat grid suits uniform DDA marching; a sparse octree suits hierarchical ray traversal; a VDB suits volume rendering in VFX pipelines. Those storage choices are covered in the storing domain; the voxel pipeline page ties them together.

---

## building-block pages

Each path has its own page that goes deeper:

- [grid ray traversal](./grid-ray-traversal.md) — DDA stepping through a uniform grid
- [sparse voxel octree raytracing](./sparse-voxel-octree-raytracing.md) — hierarchical traversal, the Laine & Karras approach
- [volume ray casting](./volume-ray-casting.md) — the emission-absorption integral
- [splatting and point rendering](./splatting-and-point-rendering.md) — object-order, forward projection
- [choosing a render path](./choosing-a-render-path.md) — the full decision matrix

---

## references

[1] Laine, S. and Karras, T. (2010). "Efficient Sparse Voxel Octrees." *Proceedings of the 2010 ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games*, pp. 55–63. DOI: 10.1145/1730804.1730814. [local PDF](../papers/laine-karras-2010-efficient-sparse-voxel-octrees.pdf) · [source](https://dl.acm.org/doi/10.1145/1730804.1730814)

[2] Fang, Z., Luo, Y., Miao, C., and colleagues (2025). "Aokana: A GPU-Driven Voxel Rendering Framework for Open World Games." *arXiv:2505.02017*. (Demonstrates SVDAG-based ray marching at tens of billions of voxels in real time.) [local PDF](../papers/fang-2025-aokana-gpu-driven-voxel-rendering.pdf) · [source](https://arxiv.org/abs/2505.02017)

[3] Kaufman, A. (1994). "Volume Visualization: Principles and Advances." *ACM SIGGRAPH Course Notes*. (Foundational treatment of image-order vs. object-order volume rendering and the emission-absorption model.) [local PDF](../papers/kaufman-1993-volume-visualization-principles-advances.pdf) · [source](https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf)
