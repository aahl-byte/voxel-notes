<link rel="stylesheet" href="./css/globals.css">

# baking ambient occlusion and light

You want voxels that look solid and grounded — corners that feel recessed, undersides that sit in shadow, surfaces that read as three-dimensional even though every face is a flat quad. The standard approach costs almost nothing at runtime: during mesh construction you compute per-vertex shading values from the surrounding voxels and write them directly into the mesh as vertex attributes. The GPU interpolates across each quad for free. No ray tracing, no shadow maps, no extra draw passes — the shading was decided once, at mesh time.

This is the technique Minecraft popularized as "smooth lighting," and Mikola Lysenko's 2013 article [1] gave it a name and a clean algorithm that became the community standard. It has two parts that are usually applied together: <em>ambient occlusion</em> to darken crevices and corners, and <em>smooth lighting</em> to blend block-light and skylight across face boundaries. Together they account for most of the visual weight that makes a blocky voxel world feel inhabitable rather than flat.

---

## the coarse model — why it works

Ambient light in the real world comes from every direction at once — sky, bounce off walls, scatter through dust. Corners and crevices receive less of it because solid surfaces block some of those incoming directions. That subtle darkening at edges is what tells your eye a shape has volume.

In a voxel world you can fake this cheaply because the geometry is on a grid. For any vertex on a face, there are exactly three neighboring voxels that can block ambient light reaching that corner: the two voxels adjacent along the face edges, and the diagonal voxel behind the corner. Count how many of those three are solid, and you have an occlusion level. Four possible values (0, 1, 2, or 3 solid neighbors) map to four brightness levels. Computed at mesh time, stored in the vertex, done.

A beginner can stop here and already hold the true model: **count the solid neighbors at each vertex corner, darken accordingly, store the result in the mesh**.

---

## ambient occlusion — the per-vertex voxel algorithm

### what the neighbors represent

Every visible face has four vertices. Each vertex sits at the intersection of that face with three neighboring voxels: two that share an edge with the face ("side1" and "side2") and one diagonal corner voxel behind the vertex. Those three voxels are the only ones that matter for that vertex's occlusion — anything farther away contributes negligibly at voxel scale.

The geometry is always the same regardless of which face you're on; only which world-space offsets you sample changes per face direction. Lysenko's article [1] gives the complete offset tables for all six face orientations.

### the formula

Given the three binary neighbor samples — each is 1 if solid, 0 if empty — the occlusion level for that vertex is [1]:

```
function vertexAO(side1, side2, corner):
    if side1 AND side2:
        return 0          // fully occluded: both edges blocked
    return 3 - (side1 + side2 + corner)
```

The result is an integer 0–3, where 3 is unoccluded (full brightness) and 0 is maximum shadow. The special case for `side1 AND side2 == true` short-circuits the corner: when both edge-neighbors are solid, light cannot reach the corner at all, so the corner's own value is irrelevant — it is already fully blocked.

These four levels are then mapped to brightness multipliers. Lysenko's reference implementation uses `[0.0, 0.25, 0.5, 0.75]` scaling applied per vertex. The Exile engine [2] applies a gentler curve `[0.75, 0.825, 0.9, 1.0]` to keep even fully-occluded corners readable.

### the quad-flip rule

When you split a quad into two triangles, the GPU interpolates each triangle's vertex values across its area using barycentric coordinates. The problem: the two triangles share a diagonal edge, and which diagonal you choose determines how the shading gradient runs. If you always split the same way, you get visible anisotropy — a diagonal seam where the interpolated shading disagrees across the two triangles.

The fix is to choose the diagonal based on the AO values themselves [1]. Label the four vertex AO values `a00, a01, a11, a10` in clockwise order around the quad. Then:

```
if (a00 + a11) > (a01 + a10):
    emit flipped quad     // diagonal runs the other way
else:
    emit normal quad
```

This rule places the shared edge along the steeper gradient, minimizing the visual discontinuity. Without it, diagonally shaded corners produce a telltale streak. With it, the interpolation chooses the split that makes both triangles agree as closely as possible. The Exile pipeline [2] also stores all four AO values in each vertex and re-interpolates them manually in the fragment shader using UV coordinates — a more expensive approach that eliminates the artifact entirely at the cost of extra per-vertex data.

The quad flip interacts with meshing: see [blocky and greedy meshing](../meshing/blocky-and-greedy-meshing.md) for how greedy merging of coplanar faces must preserve per-vertex AO values independently — a merged quad spanning multiple voxels cannot re-use a single AO value.

---

## baking — storing shading in the mesh

Computing AO per-vertex at mesh time and writing the result into a vertex attribute is called <em>baking</em>. The word comes from offline rendering — where you "bake" a complex light simulation into a texture once, then use the texture at no extra cost at runtime. The voxel per-vertex approach is the same idea at a smaller scale: move the computation to mesh time so the GPU does nothing at draw time beyond interpolating the already-computed values.

What gets stored varies by implementation:

- **a single packed integer (0–3)** per vertex, consuming 2 bits — the minimum for four levels
- **a float (0.0–1.0)** per vertex, for direct use as a brightness multiplier in the shader
- **all four face-corner AO values packed into each vertex** [2], allowing the fragment shader to re-interpolate manually and sidestep the triangle-split anisotropy problem
- **a lightmap texture** — a per-face or per-chunk 2D texture carrying baked light values, sampled in the shader; costs texture memory and requires UV generation at mesh time, but offers higher resolution than per-vertex can provide

The shader cost is identical across all these strategies: a single multiply or texture sample in the fragment stage. The CPU cost — computing the neighbor lookups and packing the values — is absorbed into mesh generation, which already needs to visit every voxel's neighbors to decide face visibility.

### the cost: re-baking on edits

Baked data has one real cost: it must be recomputed whenever the voxel data changes. In a chunk-based engine, placing or removing a single block marks its chunk — and potentially its neighboring chunks, if the changed voxel sits on a chunk boundary — as dirty and schedules a re-mesh. The re-mesh recomputes AO and light for every visible face in the chunk from scratch. This is why the [threading and meshing pipeline](../engines/threading-and-meshing-pipeline.md) matters: re-meshing must happen off the main thread to avoid frame hitches.

The tradeoff is straightforward. A dirty-chunk system means that edits cost a delayed re-mesh instead of immediate per-frame computation. For most voxel games — where edits are player-paced events, not every-frame changes — this is the right side of the tradeoff. See [the performance budget](./the-performance-budget.md) for how baking fits into the broader chunk lifecycle.

---

## smooth lighting — blending block-light and skylight at vertices

Ambient occlusion darkens geometry based on local solid neighbors. Smooth lighting is the complementary technique: instead of flat per-face shading from a single block's light value, you average the light values of the voxels around each vertex to produce a smoothly interpolated gradient across each face.

### how averaging works

Each vertex on a visible face is adjacent to up to four voxels in the plane of that face. For each vertex, sample the light value of each of those four surrounding voxels and average them [3]. The resulting per-vertex light value, when interpolated by the GPU across the quad, produces a smooth gradient — the same gradient Minecraft's smooth lighting mode creates.

A vertex where one of its four surrounding voxels is solid gets a lower average (one sample contributes 0 or near-0), producing natural shadow at edges and inside corners. This means smooth lighting and per-vertex AO produce overlapping effects: the AO formula captures the three-neighbor geometric occlusion, while the light average captures how much propagated light is actually reaching that corner. Both are applied multiplicatively in the fragment shader.

### skylight and block-light channels

The [light propagation](../simulation/light-propagation.md) page covers how these values are computed and stored per-voxel. The meshing pass consumes those already-propagated values. Most voxel engines track two separate channels [4]:

- **skylight** — how exposed a voxel is to the sky, decremented by each opaque block between it and the top of the world
- **block-light** — emitted by torches, lava, and other light sources, propagated by flood-fill with per-step falloff

At mesh time, each channel is averaged independently across the four voxels surrounding each vertex. The fragment shader combines them — typically `max(skylight × sky_factor, block_light)` — so that sunlight and artificial light don't cancel each other out.

### the link to light propagation

Smooth lighting at mesh time is downstream of light propagation in the simulation pass. The light propagation flood-fill must complete for a chunk — and its neighbors, since vertex averaging reaches across chunk boundaries — before the mesher can compute valid per-vertex light values. This creates a data-dependency ordering that the [threading and meshing pipeline](../engines/threading-and-meshing-pipeline.md) must respect: propagate light, then mesh.

---

## where baking stops and real-time GI begins

Baked AO and smooth lighting are computed from static geometry. They cannot represent:

- dynamic light sources that move (a carried torch, an explosion flash)
- light bouncing off dynamic geometry
- shadows cast by moving objects

For those effects, you need something computed per frame. The spectrum from cheapest to most physically accurate:

| technique | cost | what it captures |
|---|---|---|
| baked per-vertex AO | once, at mesh time | static geometric occlusion |
| baked smooth lighting | once, at mesh time | static propagated light, sky/block channels |
| dynamic point lights (additive) | per light per frame | local illumination, no bounce |
| light propagation volumes | per frame, on voxel grid | one-bounce diffuse GI, approximate |
| voxel cone tracing | per frame, on sparse octree | multi-bounce diffuse + specular GI |

<em>Voxel cone tracing</em>, introduced by Crassin et al. in 2011 [5], traces cone-shaped queries through a mipmapped voxel representation of the scene to estimate indirect illumination. It can handle dynamic scenes by re-voxelizing geometry each frame before tracing, giving real bounce lighting at interactive frame rates. This is the technique explored in [voxel global illumination](../applications/voxel-global-illumination.md). It operates at a completely different cost tier from baking: GPU-heavy every frame, rather than CPU-light at mesh time.

For most blocky voxel games, baking gets you most of the visual benefit at a fraction of the runtime cost. The choice is not "baked or GI" — it is "baked as the baseline, optionally augmented by dynamic lights for moving sources."

### baked vs dynamic — when to use each

| | baked AO + smooth light | real-time GI (cone tracing) |
|---|---|---|
| runtime GPU cost | near zero (interpolation only) | high — cone queries per pixel |
| dynamic light sources | not supported | supported |
| moving geometry | requires re-mesh | immediate |
| visual quality | soft corners, smooth gradients | physically accurate bounce, color bleeding |
| implementation complexity | low — neighbor lookups at mesh time | high — voxelization + SVO + shader pipeline |
| when to use | games with player-paced edits | cinematic / high-fidelity scenes |

For further rendering context see [ways to render voxels](../rendering/ways-to-render-voxels.md).

---

## putting it together — the implementation sketch

For each visible face, for each of its four vertices, sample three neighbors, compute `vertexAO`, average surrounding light values, and pack everything into the vertex before emitting it to the GPU buffer. Then apply the quad-flip rule before writing indices.

In pseudocode, the per-vertex loop for one face:

```
for each vertex v in [bottom-left, bottom-right, top-right, top-left]:
    (side1, side2, corner) = sample_neighbors(v, face_direction)
    ao[v] = vertexAO(side1, side2, corner)
    light[v] = average_light(surrounding_4_voxels(v, face_direction))

// choose quad orientation to minimize AO interpolation artifact
if (ao[0] + ao[2]) > (ao[1] + ao[3]):
    emit_quad_flipped(vertices, ao, light)
else:
    emit_quad_normal(vertices, ao, light)
```

The neighbor offsets for `sample_neighbors` depend on which of the six face directions you're meshing; Lysenko's article [1] has the full lookup tables. The result — AO level 0–3 and a light float per vertex — is written into vertex attributes alongside position and texture coordinates. The fragment shader multiplies the interpolated values together and applies them as a darkening factor over the base color or texture.

---

## references

[1] Lysenko, M. (2013). "Ambient occlusion for Minecraft-like worlds." 0fps blog. [source](https://0fps.net/2013/07/03/ambient-occlusion-for-minecraft-like-worlds/)

[2] thenumb.at (2022). "Exile: Voxel Rendering Pipeline." Dev article. [source](https://thenumb.at/Voxel-Meshing-in-Exile/)

[3] pixelwight (2015). "Procedural Terrain: Voxel-Based Lighting." Dev blog. [source](http://pixelwight.blogspot.com/2015/06/voxel-based-lighting.html)

[4] Lysenko, M. (2018). "Voxel lighting." 0fps blog. [source](https://0fps.net/2018/02/21/voxel-lighting/)

[5] Crassin, C., Neyret, F., Sainz, M., Green, S., and Eisemann, E. (2011). "Interactive Indirect Illumination Using Voxel Cone Tracing." *Computer Graphics Forum*, 30(7), 1921–1930. DOI: 10.1111/j.1467-8659.2011.02063.x. [local PDF](../papers/crassin-2011-voxel-cone-tracing.pdf) · [source](https://research.nvidia.com/publication/2011-09_interactive-indirect-illumination-using-voxel-cone-tracing)

[6] Blunt, A. (2023). "Vertex Ambient Occlusion for Voxel Games — The Principle and Implementation." Medium. [source](https://medium.com/@andrebluntindie/vertex-ambient-occlusion-for-voxel-games-the-principle-and-implementation-e5340bd62845)

[7] Penmatsa, R. and Wyman, C. (2010). "Voxel-space ambient occlusion." *Proc. ACM I3D 2010*. [source](https://cwyman.org/abstracts/i3d2010_voxelAO.pdf)
