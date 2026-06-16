<link rel="stylesheet" href="./css/globals.css">

# blocky and greedy meshing

You want a Minecraft-style world: hundreds of millions of voxels, players carving and placing blocks in real time, the whole thing running at sixty frames a second. The visual language is intentional — crisp, cubic, readable. The engineering problem is that a naively-built cube mesh would choke any GPU before the player loaded their first chunk. Getting from raw voxel data to a mesh that ships, at that scale, is what this page is about.

The solution unfolds in two steps. First, throw away every face you do not need. Then merge what is left into the fewest possible quads. Each step is a separate, stackable optimization. You can stop after the first and have a working engine; the second buys you a significant additional geometry reduction on chunky terrain.

For the question of why you need a mesh at all instead of rendering voxels directly, see [why mesh voxels](./why-mesh-voxels.md). For how chunk data is structured before meshing begins, see [voxel data models](../foundations/voxel-data-models.md).

---

## the naive starting point

Take the simplest possible approach: for every solid voxel, emit six quads — one for each face of the cube. That is two triangles per face, twelve triangles per voxel. A single 32×32×32 chunk with every voxel filled produces 32×32×32×6 = 196,608 quads, almost all of them buried deep inside the solid mass where no eye will ever see them.

This is the baseline everyone improves from. It is correct, it is simple to implement, and it is wildly wasteful.

---

## the first essential cut — face culling

Look at any two adjacent solid voxels. The face between them is completely hidden: one voxel's +X face is glued directly to the other's −X face, with no air gap, no way for a camera ray to reach either. Emitting those faces spends triangles on geometry that cannot possibly contribute to the image.

The fix is to check, before emitting each of the six possible faces of a voxel, whether the neighbor in that direction is also solid. Only emit the face if the neighbor is empty (air, water, or any transparent voxel type). A face shared between two solid voxels never gets emitted. The only faces that survive are the ones on the surface — where a solid voxel abuts empty space.

That surviving set of faces is what the algorithm produces: a shell exactly matching the visible surface of the geometry. This is <em>face culling</em>, sometimes called hidden-face removal.

The payoff for a solid cube is dramatic. An 8×8×8 solid block of 512 voxels has a surface of 384 faces — face culling emits 384 quads instead of the 3,072 the naive approach would. An 8× reduction from one simple neighbor check per face [1].

### the six-neighbor test in pseudocode

```python
for each voxel (x, y, z) that is solid:
    for each face direction (±X, ±Y, ±Z):
        nx, ny, nz = neighbor in that direction
        if neighbor(nx, ny, nz) is empty:
            emit_face(x, y, z, direction)
```

The neighbor lookup is a constant-time array offset — see [voxel data models](../foundations/voxel-data-models.md) for why that offset is trivially cheap on a dense grid.

### chunk boundaries

Culling needs the neighbor state for voxels at the edge of a chunk. The standard approach is to read a one-voxel border of neighboring chunk data before meshing starts — an 18×18×18 working volume for a 16×16×16 chunk — so every voxel inside the chunk has a complete set of six neighbors available without any special-case logic at the edge.

---

## the second cut — merging what remains

After face culling, the surface mesh is correct but still fine-grained: every surviving face is its own quad, sized exactly one voxel. A large flat wall of identical stone blocks produces one quad per block, even though the entire wall could be described as a single large rectangle.

The observation is that adjacent coplanar faces of the same voxel type can be merged into a single, larger quad without changing what the surface looks like. A 16×16 stone wall becomes one quad instead of 256. The algorithm that does this merge is <em>greedy meshing</em>, introduced by Mikola Lysenko in a 2012 blog post that became the canonical reference for voxel meshing [1].

### how the sweep works

Greedy meshing processes the volume one axis-aligned slice at a time — think of cutting the chunk into 32 horizontal layers, then 32 front-to-back slices, then 32 left-to-right slices, for each of the three face orientations.

Within a single slice (say, all the top faces of voxels at height y=5):

1. **Build a mask.** Scan across the 2D grid of this slice. For each cell, record whether a face should be emitted here — meaning the voxel below is solid and the voxel above is empty — and what type that face is.

2. **Find the widest possible quad.** Scan through the mask cell by cell. When you find an unprocessed face at position (u, v), scan rightward as far as cells match in type and haven't been merged yet. That gives the maximum width w.

3. **Extend downward.** Try to extend the w-wide strip downward row by row, as long as all w cells in each new row are the same type and unprocessed. That gives the maximum height h.

4. **Emit a single quad** covering the w×h rectangle, mark all those cells as processed, and continue scanning from the next unprocessed cell.

The result for an 8×8 flat face — where the naive approach would emit 64 quads and face culling would also emit 64 — is a single quad. For an 8×8×8 solid cube, greedy meshing emits exactly 6 quads: one per face, each covering the full 8×8 extent [1].

On real terrain the reduction is less extreme because block types interrupt the merge. A noisy terrain example measured by Lysenko went from 2,198 quads (naive) to 1,670 quads (greedy) — a meaningful reduction, and the savings grow sharply on the flat, large-area surfaces common in hand-built structures [1].

### a sketch of the slice sweep

```
Slice (top faces at y = 5):

  [stone][stone][stone][air  ][air  ]
  [stone][stone][stone][dirt ][dirt ]
  [air  ][air  ][stone][dirt ][dirt ]

Mask (faces to emit, typed by block):
  [S    ][S    ][S    ][     ][     ]
  [S    ][S    ][S    ][D    ][D    ]
  [     ][     ][S    ][D    ][D    ]

Greedy sweep emits:
  one 3×2 stone quad (top-left block of stone)
  one 1×1 stone quad (isolated stone at row 3)
  one 2×2 dirt quad
  → 3 quads instead of 7
```

Each quad is emitted by finding the largest rectangle of matching, unprocessed mask cells using the greedy scan [1].

---

## the catches greedy meshing creates

Merging faces into large quads buys geometry savings but introduces two complications that a face-culling-only mesh does not have.

### texture tiling across merged quads

A single-voxel face gets UV coordinates of (0,0) to (1,1) — the full texture fills the face. A merged quad covering N voxels in each direction needs the texture to tile N times across it. Simply stretching the coordinates from 0 to N gives you tiling if the hardware wraps correctly, but a texture atlas (all block types in one image) makes wrapping much harder: a standard atlas subtexture does not wrap at its boundary, it bleeds into the neighboring tile.

There are two common solutions:

- **Array textures** (`GL_TEXTURE_2D_ARRAY` / `TEXTURE_2D_ARRAY` in modern APIs) — each block type is a separate layer in an array. The UV coordinates can grow freely (0 to N) and the hardware tiles within the layer without any boundary problem. This is the clean path when the target platform supports it [2].
- **Shader-side modulo** — pass the quad dimensions as extra vertex data, and in the fragment shader compute `uv mod tile_size` to re-map coordinates back into the single-tile range before sampling the atlas. Requires extra texture lookups near mip boundaries to avoid the grey-bar artifact caused by the GPU's LOD gradient calculation failing at wrap points [2].

Both paths work; array textures are simpler to implement correctly.

### ambient occlusion at merged corners

Voxel worlds almost universally compute <em>per-vertex ambient occlusion</em> (AO) — at each vertex of the mesh, count how many neighboring solid voxels block ambient light and darken the vertex accordingly. The GPU interpolates AO across each triangle during rasterization, producing the characteristic soft darkening in corners that makes blocky geometry feel grounded. For the full treatment of how AO is baked and what it costs, see [baking ambient occlusion and light](../optimization/baking-ambient-occlusion-and-light.md).

Greedy meshing creates two distinct AO complications.

**Merging only valid when AO matches.** A merged quad must have identical AO values at the vertices along its merged edges; otherwise the interpolation will be discontinuous across the seam. In practice this means faces can only be merged if they share the same per-corner AO pattern — an additional constraint in the face-matching test [3].

**Quad diagonal orientation matters.** Each quad is rasterized as two triangles, split along one diagonal. When the four corner AO values are unequal, the choice of which diagonal to split along changes how the shading gradient appears. The rule: split so the diagonal faces toward the darker pair of corners. Comparing `min(ao₀, ao₃)` vs `min(ao₁, ao₂)` tells you which diagonal is correct. Getting this wrong produces a visible stripe artifact across the face [3].

Both issues are well-understood and solvable within the greedy algorithm; they just require that the face-matching and quad-emission logic carry AO values alongside block type, and that the triangulation order is computed per quad rather than assumed constant.

---

## when to use blocky meshing — and when not to

Greedy meshing is the right algorithm for a specific class of problem. The choice matters enough to be explicit about.

### blocky is the right call when

- The aesthetic is intentionally cubic. The Minecraft look is a feature, not a limitation — its readability and handcraft quality are part of the design.
- Players edit the terrain. A blocky mesh re-meshes a chunk in milliseconds; the pipeline for smooth meshers (marching cubes, surface nets, dual contouring) is more complex and harder to parallelize at interactive rates.
- Block types have hard boundaries. Greedy meshing represents each block face as a discrete unit, which maps cleanly onto discrete block semantics.
- You need a simple, debuggable pipeline. Face culling followed by greedy meshing is well-understood, widely implemented, and easy to profile. For the threading and scheduling side of the pipeline see [threading and meshing pipeline](../engines/threading-and-meshing-pipeline.md).

### blocky is the wrong call when

- The geometry needs smooth, organic surfaces — terrain with rounded hills, character bodies, fluid volumes. Blocky meshing cannot express sub-voxel surface detail; the staircase artifact at diagonal edges is unavoidable.
- The voxel data carries a continuous density or distance field (see [voxel data models](../foundations/voxel-data-models.md)). That data is the input to [marching cubes](./marching-cubes.md), [surface nets](./surface-nets-and-dual-contouring.md), or similar smooth extractors — extractors that produce a surface that follows the density isosurface rather than snapping to grid boundaries.
- Triangle count on the GPU is the bottleneck and the geometry is highly curved. Greedy meshing only merges *flat* coplanar runs; on noisy terrain the reduction is modest. GPU-side techniques can sometimes reduce draw costs without changing the mesh at all — see [GPU voxel techniques](../optimization/gpu-voxel-techniques.md).

| | face culling only | face culling + greedy | smooth mesher (marching cubes etc.) |
|---|---|---|---|
| visual style | blocky, 1-voxel quads | blocky, large merged quads | smooth, curved |
| quad count (flat surfaces) | proportional to surface area | very low | moderate to high |
| quad count (noisy terrain) | high | moderate reduction | moderate |
| texture handling | trivial | needs array textures or shader trick | trivial |
| per-vertex AO | straightforward | needs matched AO + quad flip | straightforward |
| mesh build time | fast | moderate (3× face-cull time) | higher |
| edit re-mesh cost | low | low-to-moderate | moderate-to-high |
| sub-voxel surface detail | none | none | yes |

For a side-by-side of the full meshing landscape — including surface nets and dual contouring — see [choosing a meshing algorithm](./choosing-a-meshing-algorithm.md).

---

## specifics

### what to compare when merging faces

Two faces can be merged in greedy meshing only if all of the following match:

- same block type (same texture / material)
- same face normal direction (already guaranteed within a slice)
- same per-vertex AO values along the shared edge
- same lighting / tint data if baked into vertices

Any mismatch on any attribute breaks the merge — the mask entry is treated as a different type and starts a new quad.

### chunk size and sweep count

For a C³-voxel chunk, the full greedy sweep processes 3 × C slices (one axis per orientation, C slices per axis). Each slice is a C×C 2D pass over the mask. Total work is O(C³) — linear in the number of voxels — the same asymptotic cost as face culling alone, though with a larger constant factor.

### binary greedy meshing

A modern variant encodes the voxel occupancy in bitmasks (one bit per voxel per column). The face-exposure test becomes a bitwise XOR-and-shift operation that identifies all exposed faces in 64 columns simultaneously. The subsequent greedy merge runs over the resulting bitmask rows. This approach can reduce meshing time by 3–5× over the naive per-voxel loop on large chunks, at the cost of additional implementation complexity. See the cgerikj/binary-greedy-meshing repository for a reference implementation.

---

## references

[1] Lysenko, M. (2012). "Meshing in a Minecraft Game." *0fps blog*. [source](https://0fps.net/2012/06/30/meshing-in-a-minecraft-game/)  
(The canonical description of face culling and greedy meshing for voxel engines, including quantitative quad-count comparisons and the 2D sweep algorithm.)

[2] Lysenko, M. (2013). "Texture Atlases, Wrapping and Mip Mapping." *0fps blog*. [source](https://0fps.net/2013/07/09/texture-atlases-wrapping-and-mip-mapping/)  
(Covers the UV tiling problem on merged greedy quads, the atlas boundary-bleed issue, the 4-tap mip-map fix, and array textures as the clean alternative.)

[3] Lysenko, M. (2013). "Ambient Occlusion for Minecraft-Like Worlds." *0fps blog*. [source](https://0fps.net/2013/07/03/ambient-occlusion-for-minecraft-like-worlds/)  
(Per-vertex AO calculation for voxels, the quad-diagonal flip rule, and how AO values constrain face merging in greedy meshing.)
