<link rel="stylesheet" href="./css/globals.css">

# lod and culling

A voxel world large enough to be interesting — cliffs, caves, cities, open plains stretching to the horizon — contains far more geometry than any GPU can shade in 16 milliseconds. The art of staying at 60 fps is not making the world smaller; it is making the GPU only do the work that actually changes a pixel the player sees.

Two families of technique do this. The first family says **don't draw it at all**: test each chunk before it ever reaches the rasterizer and skip it if it provably contributes nothing to the frame. The second family says **draw it cheaper**: replace distant chunks with coarser geometry so that geometry-processing cost falls off with distance instead of staying flat. Both operate at the level of whole chunks (not individual voxels), and both pay a small test cost upfront to save a much larger shading cost later.

Before reaching for either, know your real bottleneck — see [the performance budget](./the-performance-budget.md). Culling and LOD fix geometry and draw-call overhead; they cannot rescue a scene that is fill-rate or memory-bandwidth limited for other reasons.

---

## the three "don't draw it" tests

Each of the three tests below eliminates chunks that provably cannot contribute visible pixels. They are ordered cheapest-to-most-expensive and are typically applied in sequence: a chunk that fails an early test never reaches the later ones.

### outside the camera's view volume — frustum culling

The camera sees a truncated pyramid of space: wide at the far end, narrow at the near end, bounded on all six sides by planes. Anything outside that pyramid cannot appear on screen.

Before sending a chunk to the GPU, compute its <em>frustum culling</em> result by testing its bounding box against the six planes of the view frustum. If the box lies entirely on the wrong side of any one plane, the whole chunk is outside the frustum and can be skipped with no further work.

**The test in concrete terms.** A chunk's axis-aligned bounding box (AABB) is two points: a minimum corner and a maximum corner. For each of the six frustum planes, find the corner of the box that is most in the plane's positive direction (the "p-vertex") and the corner most in the negative direction (the "n-vertex"). If the p-vertex is on the negative side of any plane, the entire box is outside — reject it immediately. [1]

- Fast: six dot products per chunk, easily run for thousands of chunks on the CPU.
- Conservative: a box that passes all six tests might still be mostly off-screen (no false rejections, possible false passes).
- Chunk granularity: test the chunk's AABB, not individual voxels.
- Tighter bound: if the chunk has sparse geometry, use the tightest AABB that wraps only non-empty voxels rather than the full chunk extent — more culled chunks at modest cost.

For voxel engines with millions of chunks, frustum culling can be moved entirely to a GPU compute shader — see [the GPU techniques page](./gpu-voxel-techniques.md) — where all chunks are tested in parallel with no CPU involvement [3].

### hidden behind nearer geometry — occlusion culling

A chunk that passes the frustum test might still be completely hidden behind mountains or buildings closer to the camera. No pixel of the chunk will survive depth testing; the GPU will shade it and then discard every result.

The general problem of knowing "is this object hidden behind other objects?" is called <em>occlusion culling</em>. Two practical approaches exist for voxel renderers.

#### hardware occlusion queries

A GPU occlusion query renders a cheap proxy for the chunk (often its AABB as six triangles) with depth writing disabled, and reports back how many fragments passed the depth test. If zero fragments passed, the chunk is fully occluded and can be skipped. If any passed, the chunk is at least partially visible.

The catch is latency: the CPU must wait for the GPU result before deciding what to draw, which stalls the pipeline. Coherent Hierarchical Culling (CHC++) addresses this by exploiting frame-to-frame coherence — visibility rarely flips from one frame to the next — so the algorithm assumes most chunks stay in the same state and only issues queries for chunks near the visibility boundary, batching those queries to hide latency [2].

#### hierarchical Z-buffer (HZB)

A more GPU-native approach builds a depth pyramid: after rendering a first pass of visible geometry, generate a mip chain from the depth buffer where each level stores the *maximum* depth of the four texels beneath it. The coarsest mip is a single pixel holding the farthest depth in the whole frame.

To test a chunk for occlusion: project its AABB onto the screen to get a rectangle, pick the mip level at which one 2×2 block of texels covers that rectangle, sample the maximum depth at that block, and compare to the chunk's minimum depth. If the chunk's nearest point is farther than the sampled maximum depth, the chunk is behind everything in that region — skip it [4].

This is called <em>hierarchical Z-buffer</em> (HZB) culling. The whole test runs in a compute shader, no CPU readback needed, and the depth pyramid costs one extra downsampling pass per frame.

**Two-pass culling** combines both strengths: a first pass renders whatever was visible last frame to build a good depth pyramid cheaply; a second pass tests newly visible candidates against that pyramid [5]. Objects that were visible last frame are assumed still visible (temporal coherence) and drawn immediately; only the uncertain candidates pay the query cost.

#### when to use which

| situation | reach for |
|---|---|
| moderate scene depth, CPU budget available | hardware occlusion queries + CHC++ |
| GPU-driven pipeline, many occluded chunks | HZB culling in compute |
| hybrid (typical open-world voxel game) | two-pass: first-pass from last frame → HZB → second-pass queries |
| very open scene (sky visible everywhere) | skip occlusion culling — frustum culling is sufficient |

### facing away — backface culling

A solid voxel chunk has faces pointing in all six directions. The faces pointing away from the camera — whose normals point into the viewport rather than toward the camera — cannot be seen, and the rasterizer can discard them for free with standard backface culling enabled.

More importantly for voxel engines: <em>per-face culling during meshing</em> already removes far more than rasterizer backface culling alone. When [blocky or greedy meshing](../meshing/blocky-and-greedy-meshing.md) is built, any face shared between two solid voxels is already omitted from the mesh — it can never be seen from either side. The mesh that reaches the GPU already has only external faces. On closed terrain, this alone removes roughly half the raw face count before any runtime test runs.

The net result is that backface culling per se is less important in voxel pipelines than in general mesh pipelines: the meshing step effectively handles it at build time. Runtime backface culling is still worth enabling (it is free on modern GPUs), but it is not a major lever to tune.

---

## the "draw it cheaper" lever — distance LOD

Even after aggressive culling, the chunks that do pass are not all equally worth the same render cost. A chunk at the edge of the draw distance occupies four screen pixels; a chunk next to the camera fills a quarter of the screen. Spending the same geometry budget on both is wasteful.

<em>Level of detail</em> (LOD) solves this by selecting a coarser representation of a chunk as distance increases. The goal is that the chunk's on-screen contribution matches its rendering cost: far away, a coarser mesh with fewer triangles; nearby, full-resolution geometry.

### distance rings and the three representation tiers

In practice, most voxel engines divide the world into distance rings around the camera and assign a mesh quality tier to each ring:

- **Near ring (full resolution):** every voxel contributes normally-meshed faces. This is the zone the player is looking at and moving through.
- **Mid ring (coarser mesh):** the chunk is re-meshed at a lower voxel resolution (e.g., 2×2×2 voxels collapsed to one sample). Fewer triangles, visible quality drop is masked by distance. This is where LOD seams between rings appear — see [LOD seams and transvoxel](../meshing/lod-seams-and-transvoxel.md) for the stitching problem this creates.
- **Far ring (impostor or billboard):** at extreme distance, even a low-res mesh may be more than the on-screen pixel count warrants. Replace the chunk with a flat textured quad that always faces the camera — an <em>impostor</em> (or <em>billboard</em> if axis-locked rather than fully camera-facing). The texture is a pre-rendered or GPU-captured snapshot of the chunk from several angles; the GPU picks the closest-angle snapshot per frame. Cost: one quad, one texture lookup. Visual error: acceptable at extreme distance.

For smooth terrain using marching cubes or dual contouring, the LOD scheme pairs with an octree or clipmap structure: coarser cells farther from the camera, finer cells nearby. The engine-side organization of this is covered in [LOD in engines](../engines/lod-in-engines.md); the meshing-side seam problem is in [LOD seams and transvoxel](../meshing/lod-seams-and-transvoxel.md).

### blocky voxel LOD — the POP buffer approach

For Minecraft-style blocky terrain (where smooth interpolation is not wanted), a clean LOD method works by progressively rounding vertex positions to coarser grid levels. Vertices that collapse to the same point at a given LOD level produce degenerate (zero-area) faces that can be culled. The result is a single sorted buffer per chunk; runtime LOD selection is just choosing a read offset into that buffer — no multiple mesh copies needed [7].

Transitions between LOD levels use geomorphing: smoothly interpolating vertex positions as the camera crosses a LOD boundary, avoiding the sudden visual pop that would otherwise occur.

### LOD and the render path

LOD interacts with the render path choice. A mesh-rasterization pipeline plugs LOD in naturally — just swap which mesh you submit. A raymarching or SVO/DAG pipeline has its own distance-coarsening built into the tree traversal: stop subdividing earlier when the projected cell size falls below a threshold pixel size. See [choosing a render path](../rendering/choosing-a-render-path.md) for how the render path shapes your LOD options.

---

## doing culling at chunk granularity — and on the GPU

All three culling tests above apply to whole chunks, not individual voxels or triangles. This is the right granularity: testing millions of individual faces on the CPU would cost more than just drawing them.

**CPU-side chunk culling** (frustum and simple occlusion) is cheap enough to run every frame for worlds of thousands of chunks. A typical implementation:

1. Walk the chunk grid outward from the camera.
2. Frustum-test each chunk's AABB against the six planes.
3. Optionally, test against a conservative occlusion structure (e.g., a low-resolution shadow map from the previous frame).
4. Submit only surviving chunks to the GPU draw list.

**GPU compute culling** scales this to hundreds of thousands of chunks. A compute shader receives a buffer of all chunk AABBs and camera data, tests each chunk in parallel, and writes surviving chunk draw commands into an indirect draw buffer. The GPU's draw call then reads from that buffer directly — the CPU never touches the per-chunk visibility results.

```
// Sketch — one thread per chunk
uint chunkID = gl_GlobalInvocationID.x;
AABB box = chunkBounds[chunkID];

if (!frustumTest(box, frustumPlanes)) return;   // frustum reject
if (hzbOccluded(box, hzbMip, camera))  return;  // occlusion reject

// survived — emit an indirect draw command
uint slot = atomicAdd(drawCount, 1);
drawCommands[slot] = makeDrawCommand(chunkID);
```

The draw list is then consumed by a single `vkCmdDrawIndirectCount` (Vulkan) or `glMultiDrawElementsIndirectCount` (OpenGL) call. One GPU command draws every visible chunk [3][8]. The Aokana framework demonstrated this approach at scale — tens of billions of voxels, SVDAG-compressed, with compute culling and LOD — achieving up to 4.8× faster rendering than prior state-of-the-art [6].

For the full picture of what else a GPU compute pipeline can do (mesh generation, streaming, ray traversal), see [GPU voxel techniques](./gpu-voxel-techniques.md).

---

## the budget framing — test cost vs. draw cost

Culling and LOD are not free. Each test consumes CPU time (or GPU compute cycles) and the HZB pass consumes a full depth-buffer downsampling. The trade is worth it when the saved draw cost exceeds the test cost — which is almost always true for large worlds, but not guaranteed for small, open scenes.

A useful way to reason about this: [the performance budget](./the-performance-budget.md) framing tracks where your 16 ms actually goes. Culling attacks the draw-call count and triangle throughput portions of that budget. LOD attacks both triangle throughput and, for impostors, draw-call count.

**Measure overdraw to find hidden cost.** Overdraw is the number of times a screen pixel is shaded and then discarded because a nearer fragment overwrites it. A pixel shaded five times costs five times as much shader work as necessary. Most GPU profilers expose a pixel overdraw heatmap — bright hotspots in that view are where occlusion culling is failing or where the depth pre-pass is missing. In a voxel engine with good meshing, overdraw is modest (opaque geometry, front-to-back draw order helps), but dense foliage or transparent water can spike it badly.

**Summary of what each lever buys and costs:**

| technique | what it saves | what it costs | when to skip |
|---|---|---|---|
| frustum culling | draw calls, vertex processing | 6 dot products per chunk | never — it is always worth it |
| occlusion queries (CHC++) | shading of hidden chunks | query overhead, latency management | open scenes with few occluders |
| HZB culling | shading of hidden chunks | one depth-pyramid pass per frame | if no GPU compute budget |
| backface culling | ~50% of triangle rasterization | effectively free (GPU state) | never |
| distance LOD | triangles for far chunks | LOD mesh build time, seam work | tiny worlds that fit in VRAM |
| impostors | extreme-distance draw cost | impostor capture pass, blending | worlds with no far horizon |
| GPU indirect draw | CPU draw-call overhead | compute shader, buffer management | if draw count is already low |

---

## references

[1] Assarsson, U. and Möller, T. (2000). "Optimized View Frustum Culling Algorithms for Bounding Boxes." *Journal of Graphics Tools*, 5(1), 9–22. [source](https://www.cse.chalmers.se/~uffe/vfc.pdf)

[2] Mattausch, O., Bittner, J., and Wimmer, M. (2008). "CHC++: Coherent Hierarchical Culling Revisited." *Computer Graphics Forum*, 27(2), 221–230. DOI: 10.1111/j.1467-8659.2008.01119.x. [source](https://dcgi.fel.cvut.cz/home/bittner/publications/chc++.pdf)

[3] Vulkan Documentation Project. (2024). "GPU Rendering and Multi-Draw Indirect." Khronos Group. [source](https://docs.vulkan.org/samples/latest/samples/performance/multi_draw_indirect/README.html)

[4] RasterGrid. (2010). "Hierarchical-Z Map Based Occlusion Culling." RasterGrid Blog. [source](https://www.rastergrid.com/blog/2010/10/hierarchical-z-map-based-occlusion-culling/)

[5] Kruskonja, M. (2024). "Two-Pass Occlusion Culling." *Medium*. [source](https://medium.com/@mil_kru/two-pass-occlusion-culling-4100edcad501)

[6] Fang, Y. et al. (2025). "Aokana: A GPU-Driven Voxel Rendering Framework for Open World Games." *arXiv preprint arXiv:2505.02017*. [local PDF](../papers/fang-2025-aokana-gpu-driven-voxel-rendering.pdf) · [source](https://arxiv.org/abs/2505.02017)

[7] Evans, T. (2018). "A Level of Detail Method for Blocky Voxels." *0fps blog*. [source](https://0fps.net/2018/03/03/a-level-of-detail-method-for-blocky-voxels/)

[8] Momber, L. (2024). "Two-Pass Hierarchical Z-Buffer Occlusion Culling." *Medium*. [source](https://medium.com/@Lucmomber/two-pass-hierarchical-z-buffer-occlusion-culling-93171c5a9808)
