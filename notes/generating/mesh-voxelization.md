<link rel="stylesheet" href="./css/globals.css">

# mesh voxelization

You have a 3D character model, a CAD part, or an architectural scene — authored in a conventional polygon modeler as a shell of triangles. You want to bring it into a voxel pipeline so it can participate in volumetric global illumination, be simulated as a rigid or destructible body, or be prepared for 3D printing with per-cell material assignments. Mesh voxelization is the step that converts that triangle shell into a voxel representation.

The question is not just *which cells get turned on* — it is also *how many of them*, and *what you want to express*. Getting that choice right is what this page is about.

---

## the first choice: shell or solid

When you voxelize a mesh, the starting point is always the surface — a set of triangles. But there are two fundamentally different things you can ask for:

Mark only the cells that a triangle physically passes through, and stop there. The result is a thin shell of occupied voxels that follows the original surface. This is <em>surface voxelization</em>.

Mark those same cells, then also fill every cell enclosed by the surface. The result is a completely solid voxel object — every interior cell is occupied. This is <em>solid voxelization</em>.

| | surface voxelization | solid voxelization |
|---|---|---|
| cells marked | only cells a triangle intersects | surface cells plus all interior cells |
| memory | proportional to surface area | proportional to volume |
| requires watertight mesh | no | yes — interior must be well-defined |
| use case | collision shells, GI scene voxelization, thin features | 3D printing, destructibility, volumetric simulation |
| speed | faster | slower — needs an extra inside/outside pass |

**When to reach for surface voxelization:** real-time GI (you need the scene re-voxelized every frame and only surfaces emit/bounce light), collision geometry, any workflow where the interior is empty or irrelevant.

**When to reach for solid voxelization:** 3D printing (printers work with solid volumes), physics destruction (material needs to fill the interior), or any case where you will query "is this point inside the object."

---

## getting it exact: which cells does a triangle touch

The heart of surface voxelization is a precise membership test: does a given triangle overlap a given voxel cell? A voxel cell is a rectangular box; a triangle is a flat polygon. The question reduces to *do a triangle and a box intersect in 3D?*

The standard approach applies the <em>separating axis theorem</em> (SAT): two convex shapes do not intersect if there is any axis along which their projections do not overlap. For a triangle and an axis-aligned box the theorem requires checking at most 13 candidate axes:

- 3 face normals of the box (the x, y, z axes themselves)
- 1 face normal of the triangle (the plane the triangle lies in)
- 9 cross products of the triangle's three edge directions with the box's three edge directions

If all 13 projection-interval tests overlap, the triangle and box intersect — the voxel is marked. If any single test finds a gap, the shapes are separate — the voxel is skipped. Akenine-Möller formalized this into an optimized routine that is roughly 2–3× faster than prior methods for this specific test [1].

Every correct surface voxelizer runs this test (or an equivalent) to decide cell membership. The cost of doing it for every triangle–voxel pair is the reason GPU acceleration matters.

### conservative vs. thin (6-separating)

There is a subtlety in exactly which cells count as overlapping. Two common choices:

- **Conservative:** any cell the triangle touches at all — even grazing a corner or shared edge — is marked. The shell is never thinner than one voxel, never has gaps, and no part of the triangle is ever missed. This is what Schwarz & Seidel call their primary surface voxelization mode [2].
- **6-separating (thin):** a stricter test that produces a thinner shell — one that a ray traveling along any face-adjacent direction cannot pass through without hitting a marked voxel, but where corner-only grazes are not counted. Fewer cells are marked, which matters when you need a minimal watertight shell.

Conservative is the safer default. The thin variant is useful when you need the slimmest possible shell and diagonal connectivity is acceptable.

---

## the inside problem: filling the solid

Surface voxelization leaves the interior empty. To produce a solid, you need to identify which cells are inside the surface and fill them. The classic method exploits a simple topological fact: a ray cast from any outside point through a closed surface must cross the surface an even number of times. A ray from an interior point crosses an odd number of times.

So, for each column of voxels along one axis, cast a ray. Toggle a running inside/outside flag at each surface-voxelized hit. Every cell between an odd crossing and the next even one is interior. This is <em>ray parity</em> (the Jordan curve theorem generalized to 3D).

Eisemann & Décoret showed this parity computation can be done on the GPU in a single rendering pass using per-column bitmasking and XOR blending in the fragment shader — 300 000 polygons into a 1024³ grid at over 90 Hz on hardware from that era [3].

A more robust alternative replaces binary parity with the <em>winding number</em>: integrate how many times the surface wraps around a query point. The winding number gives a meaningful continuous value even on meshes with small holes or self-intersections, where ray parity can give wrong answers.

### the watertight requirement

Both ray parity and winding number assume the surface is closed — every edge shared by exactly two triangles, no gaps. A mesh with missing faces is called non-watertight (or open), and it breaks solid voxelization:

- Ray parity miscounts crossings through the gap, classifying exterior cells as interior or vice versa.
- Winding number degrades gracefully with small holes but still fails on badly open meshes.

Game-art assets frequently have holes in areas that will never be seen — the bottom of a building, the inside of a shoe. This makes naive solid voxelization unreliable for production assets without a preprocessing step to close gaps [4].

One practical fallback: render a small cube-environment view from each candidate voxel, count what fraction of rays see back-facing geometry, and mark the cell inside if most directions are enclosed. This heuristic survives moderate non-watertightness [4]. Another option: morphological flood-fill from a known exterior seed, then invert — this avoids ray casting entirely but can misclassify thin interior tunnels.

---

## doing it fast: GPU rasterization

Checking every triangle against every voxel is O(T × V). For a mesh with millions of triangles and a 512³ grid that is far too slow. The GPU offers a better path.

The insight is that the standard rasterization pipeline already solves a related problem: given a triangle in screen space, which 2D pixels does it cover? If you can reduce "which voxels does this triangle touch" to something the hardware rasterizer can answer, you get a massively parallel answer for free.

The method works like this. For each triangle, determine which of the three axis directions (X, Y, Z) gives the triangle its largest projected area — the axis most nearly perpendicular to the triangle's normal. Projecting along that axis maximizes the number of screen fragments generated and ensures no voxel the triangle passes through is missed. This choice of projection direction is <em>dominant-axis selection</em>.

A geometry shader reads each triangle's normal, finds the largest absolute component, and swizzles the vertex coordinates so that the dominant axis maps to screen-space depth and the other two axes map to screen X and Y. The hardware rasterizer then generates one fragment for every 2D grid cell the projected triangle covers. In the fragment shader, the 2D cell address plus the interpolated depth value together give the 3D voxel index, which is written atomically into a 3D texture using image load/store. Concurrent writes from multiple fragments to the same voxel use atomic operations (imageAtomicExchange or similar). This is the technique Crassin & Green describe in their OpenGL Insights chapter, using the OpenGL 4.2 image load/store and atomic counter interfaces [5].

```glsl
// geometry shader — dominant-axis selection (sketch)
vec3 n = abs(triangleNormal);
if (n.z >= n.x && n.z >= n.y) {
    // Z dominates: pass XY to screen as-is, Z becomes depth
    emit vertices unchanged;
} else if (n.y >= n.x) {
    // Y dominates: swizzle so Y becomes depth
    for each vertex v: emit(vec4(v.x, v.z, v.y, 1.0));
} else {
    // X dominates: swizzle so X becomes depth
    for each vertex v: emit(vec4(v.y, v.z, v.x, 1.0));
}
```

The hardware rasterizer does triangle setup, edge equations, and barycentric interpolation — all for free. The geometry shader just steers each triangle to the projection that maximizes its coverage, and the fragment shader does the final 3D write.

### conservative rasterization: closing the edge gaps

Standard rasterization marks a pixel only if the triangle's projected outline covers the pixel center. Thin triangles or triangles grazing a voxel boundary can miss the pixel center entirely and generate no fragment at all — leaving a hole in the voxelized surface where a triangle was present.

To prevent this, the projected triangle is dilated slightly — each edge pushed outward by half a pixel — so that any pixel the triangle touches at all will have its center covered. This dilation is <em>conservative rasterization</em>. It can be implemented in the geometry shader by offsetting vertices, or on modern hardware by enabling a built-in extension flag (`NV_conservative_raster` on NVIDIA, a standard feature in DirectX 12 and Vulkan). Conservative rasterization is the GPU equivalent of the SAT overlap test's "corner-grazing counts" rule: no voxel that a triangle physically intersects will ever be missed [2][5].

---

## single-pass voxelization for real-time GI

The dominant-axis GPU technique is fast enough that applications like voxel cone tracing re-voxelize the entire scene every frame. A global illumination renderer needs the voxel grid to reflect current lighting — dynamic lights and moving objects — so it runs the voxelization pass at the start of each frame before tracing cones through the result. See [voxel global illumination](../applications/voxel-global-illumination.md) for how the voxelized radiance is consumed downstream.

For large or complex scenes, a partial update can reduce cost: the voxelized grid is cached, and only the cells near dynamic objects are cleared and re-rendered each frame. The GPU techniques for managing those budgets are covered in [GPU voxel techniques](../optimization/gpu-voxel-techniques.md).

---

## pitfalls

### thin features

Features thinner than one voxel cell — thin walls, sharp fins, narrow cables — can be missed by surface voxelization if conservative rasterization is not active. Even with it, features smaller than half a cell width may not reliably generate coverage. The only permanent fix is a finer grid, or a hierarchical approach that allocates more resolution near thin regions.

### non-watertight meshes

Solid voxelization requires a closed surface. Non-watertight meshes produce incorrect interior fills. Options, in rough order of reliability:

1. Fix the mesh first — fill holes, weld seams. Mesh-repair tools (MeshLab, Blender, CAD repair) can close most gaps.
2. Use a winding number instead of ray parity — more tolerant of small defects.
3. Use the cube-map heuristic for game art where mesh repair is impractical.
4. Fall back to surface-only voxelization when interior fill is not needed.

### voxelization vs. SDF construction

If your goal is smooth surface reconstruction — fitting a distance field to the mesh and then extracting an isosurface — you do not want to voxelize the triangle shell into occupied/empty voxels. Instead you want to compute a <em>truncated signed distance function</em> (TSDF): the signed distance from each voxel cell to the nearest triangle surface, stored only within a thin band around the surface. That process is covered in [SDF and CSG modeling](./sdf-and-csg-modeling.md), and the closely related case of building a TSDF from depth-sensor scans is in [scanned and volume data](./scanned-and-volume-data.md).

---

## where voxelization fits

Mesh voxelization is one of several paths into the voxel domain. The full landscape — procedural generation, depth scanning, SDF/CSG construction, and mesh conversion — is mapped in [where voxels come from](./where-voxels-come-from.md). How keeping a mesh compares to converting it to voxels is covered in [voxels vs other representations](../foundations/voxels-vs-other-representations.md).

---

## references

[1] Akenine-Möller, T. (2001). "Fast 3D Triangle-Box Overlap Testing." *Journal of Graphics Tools*, 6(1), 29–33. DOI: 10.1080/10867651.2001.10487535. [local PDF](../papers/akenine-moller-2001-fast-3d-triangle-box-overlap.pdf) · [source](https://fileadmin.cs.lth.se/cs/Personal/Tomas_Akenine-Moller/pubs/tribox.pdf)

[2] Schwarz, M. and Seidel, H.-P. (2010). "Fast Parallel Surface and Solid Voxelization on GPUs." *ACM Transactions on Graphics (Proc. SIGGRAPH Asia)*, 29(6), Article 179. DOI: 10.1145/1866158.1866201. [local PDF](../papers/schwarz-seidel-2010-fast-parallel-voxelization-gpus.pdf) · [source](https://michael-schwarz.com/research/publ/2010/vox/)

[3] Eisemann, E. and Décoret, X. (2008). "Single-Pass GPU Solid Voxelization for Real-Time Applications." *Proc. Graphics Interface 2008*, 73–80. [local PDF](../papers/eisemann-decoret-2008-single-pass-gpu-solid-voxelization.pdf) · [source](https://maverick.inria.fr/Publications/2008/ED08a/solidvoxelizationAuthorVersion.pdf)

[4] Game Developer. "Opinion: Robust Inside and Outside Solid Voxelization." (2013). Retrieved June 2026. [source](https://www.gamedeveloper.com/programming/opinion-robust-inside-and-outside-solid-voxelization)

[5] Crassin, C. and Green, S. (2012). "Octree-Based Sparse Voxelization Using the GPU Hardware Rasterizer." In *OpenGL Insights*, ed. Cozzi & Riccio, CRC Press, pp. 303–319. [local PDF](../papers/crassin-green-2012-octree-sparse-voxelization-gpu.pdf) · [source](https://research.nvidia.com/labs/rtr/publication/crassin2012voxelization/)
