<link rel="stylesheet" href="./css/globals.css">

# SDF and CSG modeling

The goal is sculpting and editing 3D shapes — tools that let you push and pull clay, drill holes, merge blobs, and have the result survive meshing into smooth geometry or render directly without meshing at all.

Games like *Dreams* (Media Molecule) and *Claybook* (Second Order) built entire creation suites on this approach: an artist places a brush, the brush adds or removes volume, and the world updates in real time. What makes that possible is a specific choice about *how to represent the shape in the voxel grid* — not as a raw yes/no per cell, but as a number that encodes the distance to the nearest surface with a sign that says which side you're on. Everything else in this page follows from that one choice.

For context on where SDF-based generation fits in the broader pipeline, see [where voxels come from](./where-voxels-come-from.md). For how the SDF format compares against occupancy grids and meshes at the storage level, see [voxels vs other representations](../foundations/voxels-vs-other-representations.md).

---

## the core idea — a field of distances

Imagine you could ask any point in space: *how far away is the nearest surface of this object, and am I inside or outside?* Store the answer to that question at every cell in a voxel grid. That's the whole idea.

Cells outside the object get a positive number — the distance to the surface. Cells inside get a negative number — the depth below the surface. The magnitude is always the shortest distance to the surface; only the sign changes. The surface itself sits at exactly zero: the grid passes through zero between any cell whose value is positive and any adjacent cell whose value is negative.

That zero crossing — the set of all points where the field equals zero — is the <em>signed distance field (SDF)</em>'s surface representation. Any implicit surface described this way can be recovered by finding where the field changes sign, which is why this interface is also called an <em>iso-surface</em> at value zero. The field is usually sampled into a regular voxel grid: one float per cell, covering the volume of interest.

A reader who stops here already holds the key model: **the SDF is a 3D grid of floats, where the surface is the zero crossing and the sign tells you which side you're on.**

---

## why this beats a raw occupancy grid

An occupancy grid stores a single bit per cell: filled or empty. The surface is the boundary between filled and empty cells, but the grid doesn't know *how far* any cell is from that boundary — a cell deep inside a solid looks identical to one just barely inside.

An SDF stores the actual distance. That extra information unlocks three things:

- **Smooth meshing.** Algorithms like dual contouring can use the precise zero crossing location — interpolated between adjacent cells — to place vertices accurately and recover sharp features. A raw occupancy grid forces the vertex to snap to the cell corner. See [surface nets and dual contouring](../meshing/surface-nets-and-dual-contouring.md) for the full treatment.
- **Direct rendering.** A ray can be marched through the field using the stored distances as guaranteed safe step sizes — never overshooting the surface. This is covered in the sphere tracing section below.
- **Composable operations.** Two SDFs can be combined with simple arithmetic: the surface of the combined shape emerges automatically from the resulting numbers.

---

## primitives — shapes with closed-form SDFs

Some shapes have formulas that compute the exact signed distance from any point to their surface — no grid, no lookup, just arithmetic. These are <em>SDF primitives</em>.

The sphere is the simplest: the distance from any point `p` to the surface of a sphere centered at the origin with radius `r` is just the distance from `p` to the center, minus `r`. Positive outside, negative inside, zero on the surface.

```glsl
float sdSphere(vec3 p, float r) {
    return length(p) - r;
}
```

A box, capsule, and torus each have equally compact formulas [1]:

```glsl
// axis-aligned box with half-extents b
float sdBox(vec3 p, vec3 b) {
    vec3 q = abs(p) - b;
    return length(max(q, 0.0)) + min(max(q.x, max(q.y, q.z)), 0.0);
}

// capsule from point a to b with radius r
float sdCapsule(vec3 p, vec3 a, vec3 b, float r) {
    vec3 pa = p - a, ba = b - a;
    float h = clamp(dot(pa, ba) / dot(ba, ba), 0.0, 1.0);
    return length(pa - ba * h) - r;
}

// torus with major radius t.x, tube radius t.y
float sdTorus(vec3 p, vec2 t) {
    vec2 q = vec2(length(p.xz) - t.x, p.y);
    return length(q) - t.y;
}
```

The value of closed-form primitives is that you can evaluate them anywhere, at any resolution, without sampling a grid. You only need to sample into a voxel grid once — when you want to combine many primitives together or use a voxel-based pipeline downstream. The formulas are the source of truth.

---

## combining shapes — boolean operations via min and max

Two SDFs can be combined using simple arithmetic operations. Because the SDF at any point describes distance to the *nearest* surface, combining two SDFs is a question of which surface is closer — which is exactly what `min` and `max` answer.

Describe what each operation does, then name it:

**Joining two shapes** — take the distance to whichever surface is closer at each point. This is just `min(a, b)`. The resulting zero crossing is the outer boundary of both shapes merged together. This is <em>union</em>.

**Keeping only the overlap** — take the distance to whichever surface is *farther* at each point. Only where both fields are negative (inside both shapes) does the result stay negative. This is `max(a, b)`, or <em>intersection</em>.

**Carving one shape out of another** — flip the sign of the tool shape, then take `max`. Where you were inside the tool, the flipped value is now strongly positive, pushing the result positive (outside) even deep in the base shape. The result is `max(a, -b)`, or <em>subtraction</em>. [1]

```glsl
float opUnion(float a, float b)        { return min(a, b); }
float opIntersection(float a, float b) { return max(a, b); }
float opSubtraction(float a, float b)  { return max(a, -b); }
```

Together, these three operations form <em>constructive solid geometry (CSG)</em>: building complex shapes from primitives combined by boolean operations. In a traditional mesh-based CSG pipeline, computing the boolean of two meshes is a hard geometric problem involving polygon clipping and topology repair. With SDFs the same operations reduce to a single arithmetic expression per cell.

#### smooth blending

The sharp versions of union/intersection produce crisp joins. Often you want shapes to *melt* into each other — the way clay blobs fuse when pressed together. The trick is replacing hard `min` with a soft version that blends the two values when they are close to each other.

The <em>smooth minimum (smin)</em> [1] uses a polynomial to blend the two fields within a radius `k` of equal distance:

```glsl
float smin(float a, float b, float k) {
    k *= 4.0;
    float h = max(k - abs(a - b), 0.0) / k;
    return min(a, b) - h * h * k * (1.0 / 4.0);
}
```

The parameter `k` is the blend thickness in actual distance units — the same world units as the SDF values themselves. Outside the blend zone the function behaves exactly like hard `min`, so far-apart shapes stay sharp. This is the operation that gives sculpting tools their organic, clay-like feel.

---

## sculpting — brushes that rewrite the field

A sculpting brush is a localized SDF operation applied to the voxel grid in a region around the brush position. The brush itself is a primitive SDF — a sphere, a capsule, or a custom shape — and applying it means reading every affected cell, combining its current value with the brush's SDF, and writing the result back.

- **Adding material:** union of the current field and the brush SDF — `min(current, brush)`.
- **Removing material:** subtraction — `max(current, -brush)`.
- **Smoothing or blending:** `smin` in place of hard `min`, with `k` set to the brush softness.

Only the cells within the brush's bounding box need updating. This locality is what makes real-time interactive sculpting practical: a 1024³ world field only has a few thousand cells touched per brush stroke, not the whole grid. Claybook [2] uses a 1024×1024×512 field stored in 8-bit signed format with 5 mip levels for exactly this reason — the field is queried and updated in tight local patches.

This connects directly to the runtime editing pipeline covered in [runtime editing and CSG](../engines/runtime-editing-and-csg.md).

---

## rendering SDFs directly — sphere tracing

SDFs unlock a rendering path that doesn't require meshing at all: casting rays directly through the field.

The key insight is that the SDF value at any point is a *guaranteed safe radius* — no surface exists within that distance. A ray marcher can therefore advance the ray by the current SDF value at each step, and it is mathematically impossible to overshoot and skip through a surface undetected. At each new position it samples the field again and takes another step of that size.

Hart (1996) named and formalized this technique as <em>sphere tracing</em> [3]: at each step you are guaranteed to be inside a sphere of empty space, so the step is safe. Near a surface the steps become very small as the SDF approaches zero; in open space they are large. The ray terminates when the SDF value falls below a small threshold (surface hit) or when the ray exits the volume (miss).

```glsl
float castRay(vec3 ro, vec3 rd) {
    float t = 0.0;
    for (int i = 0; i < 128; i++) {
        float d = sdf(ro + rd * t);   // sample field at current position
        if (d < 0.001) break;         // close enough — surface hit
        t += d;                        // safe to step by the SDF value
        if (t > FAR) break;           // escaped the volume
    }
    return t;
}
```

This is the basis for the ray traversal path described in [grid ray traversal](../rendering/grid-ray-traversal.md). The advantage over mesh rendering is that curved, smoothly blended SDF shapes render perfectly at any resolution without any tessellation step. The cost is that every pixel requires many field samples, which is why the GPU parallelism of a voxel grid is essential.

---

## storage tradeoffs — the cost of keeping a full field

A naive SDF stores one float per cell for the entire volume. At 512³ resolution with 4 bytes per cell that is 512 MB. Most of that space is wasted: deep inside a solid or far in empty space, the SDF is some large constant — it holds no surface information.

Two strategies cut this down:

### narrow-band (truncated) SDFs

Store only the cells within a thin shell around the surface — say, within ±5 voxels of the zero crossing. Cells outside the band are discarded (or set to a clamped sentinel value). Curless and Levoy (1996) introduced this approach as the <em>truncated SDF (TSDF)</em> to fuse depth camera scans into a volumetric model [4]. Their local PDF is already in this repo: [local PDF](../papers/curless-levoy-1996-tsdf-volumetric-range-images.pdf).

The tradeoff: a TSDF can't answer "how far is this point from the surface" for points beyond the truncation distance. For rendering or large-step sphere tracing you need the full field. For sculpting and meshing, the narrow band is usually enough.

### adaptively sampled SDFs (ADFs)

Frisken et al. (2000) proposed storing the SDF in a hierarchical structure — coarse blocks in flat regions, fine cells only near the surface [5]. The resolution adapts to the local curvature of the SDF: near-flat areas need few samples; highly curved or detailed areas get fine samples. This is the SDF analogue of a sparse octree.

The tradeoff: ADFs give excellent compression and support smooth interpolation between samples, but the hierarchical lookup is more complex than a flat grid offset.

### sparse grids (VDB)

OpenVDB [6] provides a practical sparse grid that stores only active (non-background) voxels in a B+tree-style hierarchy. Because SDF values far from the surface are nearly constant, the background value is simply the maximum clamped distance, and only the narrow band of interesting cells is stored explicitly. VDB is now the production standard for sparse SDF storage in DCC tools and offline rendering.

#### when to reach for each

| approach | storage | fast lookup | handles edits | good for |
|---|---|---|---|---|
| full grid | O(n³) | yes — flat array | yes | prototyping, small volumes |
| narrow band / TSDF | ~O(surface area) | yes | yes, near surface | scanning, online reconstruction |
| ADF (adaptive octree) | ~O(surface area × detail) | moderate — tree traversal | harder | static assets, LOD |
| sparse (VDB) | ~O(surface cells) | moderate | good with diff tracking | production offline, large worlds |

---

## the voxel data model behind the SDF

The SDF grid follows the same data model as any scalar voxel field — one value per cell, position implicit in the index, payload type is `float32` (or `int8` for compact TSDFs). The difference is *semantic*: the payload is now a signed distance, not a color or an occupancy flag. Everything in [voxel data models](../foundations/voxel-data-models.md) applies directly.

---

## references

[1] Quilez, I. (n.d.). "3D SDF Primitives and Operations." *iquilezles.org*. [source](https://iquilezles.org/articles/distfunctions/) · Smooth minimum: [source](https://iquilezles.org/articles/smin/)

[2] Aaltonen, S. (2018). "GPU-based clay simulation and ray-tracing tech in Claybook." *Game Developers Conference 2018 (GDC 2018)*. [source](https://gdcvault.com/play/mediaProxy.php?sid=1025316)

[3] Hart, J. C. (1996). "Sphere tracing: A geometric method for the antialiased ray tracing of implicit surfaces." *The Visual Computer*, 12(10), 527–545. DOI: 10.1007/s003710050084. [local PDF](../papers/hart-1996-sphere-tracing.pdf) · [source](https://experts.illinois.edu/en/publications/sphere-tracing-a-geometric-method-for-the-antialiased-ray-tracing/)

[4] Curless, B. and Levoy, M. (1996). "A volumetric method for building complex models from range images." *Proceedings of SIGGRAPH 1996*, pp. 303–312. DOI: 10.1145/237170.237269. [local PDF](../papers/curless-levoy-1996-tsdf-volumetric-range-images.pdf) · [source](https://dl.acm.org/doi/10.1145/237170.237269)

[5] Frisken, S. F., Perry, R. N., Rockwood, A. P., and Jones, T. R. (2000). "Adaptively sampled distance fields: A general representation of shape for computer graphics." *Proceedings of SIGGRAPH 2000*, pp. 249–254. DOI: 10.1145/344779.344899. [local PDF](../papers/frisken-2000-adaptively-sampled-distance-fields.pdf) · [source](https://dl.acm.org/doi/10.1145/344779.344899)

[6] Museth, K. (2013). "VDB: High-resolution sparse volumes with dynamic topology." *ACM Transactions on Graphics*, 32(3), Article 27. DOI: 10.1145/2487228.2487235. [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf) · [source](https://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf)

[7] Ju, T., Losasso, F., Schaefer, S., and Warren, J. D. (2002). "Dual contouring of hermite data." *Proceedings of SIGGRAPH 2002*, pp. 339–346. DOI: 10.1145/566570.566586. [source](https://dl.acm.org/doi/10.1145/566570.566586)
