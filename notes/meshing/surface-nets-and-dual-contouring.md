<link rel="stylesheet" href="./css/globals.css">

# surface nets and dual contouring

CAD models have razor-sharp creases where two faces meet at an angle. Hard-surface sculpts — helmets, engine parts, stylized characters — live or die by the crispness of their edges and corners. SDF-based terrain has cliff edges that should read as geological breaks, not gentle slopes. When you extract a mesh from a voxel grid using [marching cubes](./marching-cubes.md), all of those features get softened: vertices are constrained to lie on cell edges, which forces the algorithm to approximate every sharp feature with a staircase of tiny triangles. The edges round off. The corners chamfer. The geometry loses the thing that made it interesting.

Dual methods — surface nets and dual contouring — solve this by placing mesh vertices inside cells rather than on their edges. That single structural change unlocks the ability to land a vertex exactly where a sharp feature passes through, recovering the edge or corner the scalar field describes.

---

## the coarse model — two ideas working together

Before the details, here is the mental model that everything else plugs into.

**The primal/dual distinction.** [Marching cubes](./marching-cubes.md) is a *primal* method: it reads sign changes on cell edges and places a vertex on each edge that crosses the surface, then connects those edge-vertices into triangles. The vertices live *on the cell boundary*. Dual methods invert this: instead of one vertex per crossing edge, they place *one vertex per cell* — inside it — and then form faces by connecting vertices of neighboring cells. Because the vertex is free to sit anywhere inside the cell, you can steer it toward the exact feature the surface is trying to describe. That freedom is what marching cubes lacks.

**Two levels of sophistication.** Once you accept the dual idea, two implementations diverge on how they choose the vertex location.

- The simple version places the vertex at the cell center, or at the average position of the edge crossings inside that cell. No normals required, no linear algebra. This is <em>surface nets</em> (Gibson 1998) — cheap, smooth, blobby, good for organic forms.
- The sophisticated version asks: "what point best fits the tangent planes described by the surface normals at each edge crossing?" It solves a small least-squares problem per cell. This is <em>dual contouring</em> (Ju, Losasso, Schaefer & Warren 2002) — slightly more expensive, but it reconstructs sharp edges and corners that surface nets rounds away.

A beginner can stop here and already understand why these methods exist and how they differ from marching cubes and from each other. Everything below fills in the mechanics.

---

## why marching cubes rounds off features

See [marching cubes](./marching-cubes.md) for the full algorithm. The relevant constraint here: marching cubes vertices are always placed by linearly interpolating along cell edges — between the two sample points that straddle the surface. For a perfectly flat, axis-aligned face, this works fine. But for a sharp edge where two faces meet, the vertex would need to be at the corner itself, which is inside the cell — not on its edge. Marching cubes cannot reach there. The best it can do is approximate the corner with many tiny triangles at higher resolution. Dual methods eliminate this structural constraint.

---

## the dual idea — describe it, then name it

Imagine walking through a voxel grid where each cell is either inside or outside a surface (or more generally, where the scalar field changes sign somewhere inside). For each cell that the surface passes through — call it an *active cell* — you want to place exactly one mesh vertex, somewhere inside that cell. Then, for every edge in the original grid that is crossed by the surface (meaning one endpoint is inside, the other outside), you look at the four cells that share that edge and connect their four vertices into a quad face.

That is the complete structure: one vertex per active cell, one quad per active edge, quads formed by connecting the four active-cell vertices that surround each active edge. The resulting mesh is the <em>dual</em> of the active-edge structure — it is literally the dual graph of the relevant parts of the voxel grid.

Crucially, nothing in this structure constrains *where* inside the cell the vertex sits. The topology (which cells are active, which edges are active, how they connect) is determined by the sign configuration of the scalar field. The geometry (the actual 3D coordinate of each vertex) is a separate question — and that is exactly where surface nets and dual contouring diverge.

---

## surface nets — the simplest dual method

The simplest possible choice for vertex position: put it at the center of the active cell, or compute the average of all the edge-crossing positions along the cell's 12 edges. Gibson (1998) formalized this as surface nets for binary volumetric segmentations from medical imaging — data where each voxel is simply inside or outside a region, with no surface-normal information available.

The algorithm:

- For each surface cell, place a node at the center (or average of edge crossings).
- Link each node to the nodes of its six face-adjacent surface cells.
- Optionally run an energy-minimization pass that adjusts node positions to reduce surface bending energy, subject to the constraint that each node stays within its original cell. This "elastic" relaxation smooths the mesh while preventing nodes from drifting to a different cell.

The output is a quad mesh (one quad per active grid edge). It is smooth and free of the staircase artifacts that marching cubes produces. It handles topological ambiguity cleanly — unlike marching cubes, there are no lookup-table ambiguity cases.

**What surface nets cannot do.** Because vertex placement ignores the surface normal, it cannot reconstruct a sharp feature. A cell that contains a 90-degree edge between two planar faces will get a vertex somewhere near the middle, not at the crease. The resulting mesh looks organic and rounded regardless of what the underlying shape is. Surface nets is the right choice when smooth, blob-like output is the goal — terrain with rolling hills, organic sculpts, medical segmentation meshes. It is the wrong choice when hard-surface features must survive extraction.

---

## dual contouring — steering the vertex with normals

Dual contouring (Ju et al. 2002 [1]) adds one more piece of input data per edge crossing: the surface normal at the crossing point. Together, the crossing position and the normal define a tangent plane — the plane that the surface is locally trying to describe at that point. A cell with two edge crossings has two tangent planes. A cell at a sharp corner might have six or more. The vertex placement question becomes: *where is the point that lies closest to all of these tangent planes simultaneously?*

### hermite data — what it is

Each active edge (an edge where the sign changes) carries two numbers. First, the scalar values at its two endpoints determine exactly where the surface crosses the edge by linear interpolation — this is the crossing position **p**. Second, the surface normal at that crossing point — evaluated from the gradient of the scalar field — gives the normal direction **n**. The pair (**p**, **n**) on one edge is one piece of <em>Hermite data</em>.

A cell with *k* active edges collects *k* such pairs before any vertex is placed.

### the QEF — describe it, then name it

For each edge crossing, the tangent plane passes through **p** and is perpendicular to **n**. The signed distance from any candidate point **x** to this tangent plane is `n · (x − p)`. Squaring this gives the squared distance; summing across all *k* edge crossings gives a total error that is zero only if **x** lies on every tangent plane exactly — which is generally impossible for more than two planes.

The function being minimized is:

```
E(x) = Σᵢ [ nᵢ · (x − pᵢ) ]²
```

Written in matrix form, let **A** be the matrix whose rows are the normals **n₀, n₁, … n_{k−1}**, and let **b** be the column vector whose entries are the dot products `nᵢ · pᵢ`. Then:

```
E(x) = ‖Ax − b‖²
```

This is a standard linear least-squares problem. Its minimum satisfies the <em>normal equations</em>:

```
(AᵀA) x = Aᵀb
```

This minimizer — the point that minimizes the sum of squared distances to all tangent planes — is the vertex position placed inside the cell. The function E is a <em>quadratic error function (QEF)</em>: quadratic because the distance-squared terms make it a degree-two polynomial in **x**, and it is the total error that the placed vertex incurs relative to the tangent-plane evidence in the cell.

### why sharp features survive

At a flat part of a surface, all the tangent planes inside a cell are nearly parallel — they agree on a region, and the QEF minimum lands somewhere in the middle, much like a surface-nets vertex. No special treatment is needed.

At a sharp edge, the cell contains crossing points from two faces meeting at an angle. Their normals point in different directions. The two families of tangent planes intersect along a line — the edge. The QEF minimum is pulled toward that line. At a sharp corner, three or more families of planes intersect at a point — the corner — and the QEF minimum is pulled to that point. The algorithm needs no special-case detection; the geometry of the least-squares solution automatically reconstructs the feature from the tangent-plane evidence. This is why dual contouring faithfully reproduces sharp edges and corners that marching cubes smooths away.

### the QEF in practice — stability and clamping

Two practical complications arise.

**Rank deficiency.** If all the normals in a cell point in the same direction (flat surface) or only two distinct directions (an edge, but captured from a near-parallel family), the matrix AᵀA is singular or nearly singular. The normal equations have no unique solution — the minimizer is a line or plane rather than a point. The standard fix is to solve via singular value decomposition (SVD), discarding eigenvalues below a threshold and using the pseudoinverse. When the system is rank-deficient, the SVD solution moves toward the minimum-norm solution: the point in the minimum space closest to the cell's mass point (the average of the edge-crossing positions). Some implementations use Tikhonov regularization instead, adding a small `λI` term to AᵀA to make it invertible.

**Out-of-cell vertices.** The QEF minimizer is not constrained to land inside the cell. For cells near very sharp features — a crease angle close to 180 degrees, or a corner where planes intersect far from the cell center — the minimizer can land outside the cell boundary. An out-of-cell vertex causes visual artifacts: the quad connecting four adjacent cells can become non-planar, self-intersecting, or produce inverted normals. The fix is <em>clamping</em>: after solving, constrain the vertex position to the cell's bounding box (or a slightly shrunken interior). This can blunt very acute features slightly, but eliminates geometric artifacts. More sophisticated approaches clamp to a safe interior region computed from the crossing positions.

---

## forming the mesh — quads from active edges

Both surface nets and dual contouring share the same topology-building step. The mesh connectivity is determined entirely by the grid topology and the sign configuration of the scalar field — the same information used to find active cells and active edges.

For every active edge — an edge shared by four cells where one endpoint is inside the surface and the other is outside — the four cells meeting that edge all have dual vertices. Connect those four vertices in order around the edge, forming a quad. The winding order (which vertex comes first) is determined by the sign convention: the inside-to-outside direction tells you which face orientation is outward.

The output mesh is entirely quads. One quad per active edge. The quad count roughly equals the number of surface voxel-edge crossings, which is proportional to the surface area at the grid resolution. Because each dual vertex is shared by multiple quads (as many quads as the cell has active edges, up to 12), the mesh is typically more compact than a marching cubes triangulation of the same grid.

---

## non-manifold output and manifold fixes

A mesh is <em>manifold</em> if every edge is shared by exactly two faces and every vertex has a single connected ring of faces around it. Manifold meshes are required by most downstream tools: boolean operations, subdivision surfaces, 3D printing slicers, physics engines.

Dual contouring is not guaranteed to produce manifold output. The canonical failure mode is the *hourglass*: two surfaces that touch at a single cell, or a surface that folds back on itself. Both configurations cause two separate sheets of the mesh to share a single dual vertex — the one in the touching cell. That vertex becomes non-manifold: it sits at the junction of two disconnected face rings.

Schaefer, Ju & Warren addressed this in their follow-up work on manifold dual contouring [2], which detects these sign configurations and splits the shared vertex into separate vertices for each connected sheet, at the cost of slightly more complex topology bookkeeping.

A related approach is dual marching cubes (Schaefer & Warren 2004/2005 [3]; Nielson 2004 [4]), which combines the dual placement idea with explicit lookup tables that guarantee manifold topology — trading some of the flexibility of QEF-based vertex placement for a manifold guarantee.

---

## method comparison

| | marching cubes | surface nets | dual contouring |
|---|---|---|---|
| vertex placement | on cell edges | inside cell (centroid) | inside cell (QEF minimum) |
| data required | scalar values only | scalar values only | scalar values + surface normals |
| sharp features | no — smoothed away | no — rounded | yes — reconstructed |
| output topology | triangles | quads | quads |
| manifold guarantee | yes (with flip fix) | yes | not guaranteed |
| complexity | lookup table | simple average | per-cell linear algebra |
| best fit | smooth organic shapes | smooth organic shapes, performance-critical | hard-surface, CAD, SDF sculpts |

Marching cubes outputs triangles; both dual methods output quads. Quads can be split into triangles for rendering (two triangles per quad), but they also subdivide cleanly and are preferred by CAD workflows and subdivision-surface pipelines. If your downstream tools want triangles, splitting quads is trivial; going the other direction (retriangulating a marching cubes mesh into quads) is a hard geometry-processing problem.

### when to use each

**Use surface nets** when:
- Input is binary occupancy (no normals available or meaningful)
- Output will be organically smooth — terrain, sculpts, medical meshes
- You need the fastest possible meshing with minimal memory use
- Manifold output is required without extra bookkeeping

**Use dual contouring** when:
- The scalar field has a meaningful gradient (signed distance fields, CSDs, analytically defined SDFs — see [SDF and CSG modeling](../generating/sdf-and-csg-modeling.md))
- The shape has hard edges or corners that must survive extraction — [CAD models](./why-mesh-voxels.md), hard-surface sculpts, architectural geometry
- You can afford per-cell linear algebra (still cheap — a 3×3 SVD per surface cell)
- Quad output and feature fidelity matter more than a manifold guarantee

**Use marching cubes** when:
- The toolchain expects triangles and no conversion step is acceptable
- Normals are unavailable or too costly to compute
- The well-understood lookup-table implementation is a requirement
- See [choosing a meshing algorithm](./choosing-a-meshing-algorithm.md) for a fuller decision tree

---

## where these methods fit in the pipeline

Both methods are surface extraction algorithms — they sit between the voxel data and the final mesh. They assume the voxel grid already contains a meaningful scalar field. The most natural input is a [signed distance field (SDF)](../generating/sdf-and-csg-modeling.md): the gradient is exactly the surface normal, and the zero crossing is the surface. Dual contouring was designed with SDFs in mind, and the Hermite data (crossing point + normal) falls directly out of SDF evaluation.

For the broader context — why you'd mesh a voxel grid at all, what the alternatives are — see [why mesh voxels](./why-mesh-voxels.md). For the data structures that efficiently store the scalar field being meshed, see [voxel data models](../foundations/voxel-data-models.md). If the mesh will be rendered at multiple levels of detail, the seam-handling problem that arises at LOD boundaries is covered in [LOD seams and transvoxel](./lod-seams-and-transvoxel.md).

---

## references

[1] Ju, T., Losasso, F., Schaefer, S., and Warren, J. (2002). "Dual Contouring of Hermite Data." *ACM SIGGRAPH 2002*, pp. 339–346. DOI: 10.1145/566570.566586. [local PDF](../papers/ju-2002-dual-contouring-hermite-data.pdf) · [source](https://www.cse.wustl.edu/~taoju/research/dualContour.pdf)

[2] Schaefer, S., Ju, T., and Warren, J. (2007). "Manifold Dual Contouring." *IEEE Transactions on Visualization and Computer Graphics*, 13(3), pp. 610–619. DOI: 10.1109/TVCG.2007.1012. [source](https://www.cs.wustl.edu/~taoju/research/dualsimp_tvcg.pdf)

[3] Schaefer, S. and Warren, J. (2005). "Dual Marching Cubes: Primal Contouring of Dual Grids." *Computer Graphics Forum*, 24(2), pp. 195–201. DOI: 10.1111/j.1467-8659.2005.00843.x. [local PDF](../papers/schaefer-warren-2005-dual-marching-cubes.pdf) · [source](https://www.cs.rice.edu/~jwarren/papers/dmc.pdf)

[4] Nielson, G. M. (2004). "Dual Marching Cubes." *IEEE Visualization 2004*, pp. 489–496. DOI: 10.1109/VISUAL.2004.28.

[5] Gibson, S. F. F. (1998). "Constrained Elastic Surface Nets: Generating Smooth Surfaces from Binary Segmented Data." *MICCAI 1998*, Lecture Notes in Computer Science, vol. 1496, pp. 888–898. DOI: 10.1007/BFb0056277. [source](https://link.springer.com/chapter/10.1007/BFb0056277)

[6] Schaefer, S. and Warren, J. (2002). "Dual Contouring: The Secret Sauce." Technical Report, Rice University. [local PDF](../papers/schaefer-warren-2002-dual-contouring-secret-sauce.pdf) · [source](https://people.eecs.berkeley.edu/~jrs/meshpapers/SchaeferWarren2.pdf)

[7] Schroeder, W., Tsalikis, S., Halle, M., and Frisken, S. (2024). "A High-Performance SurfaceNets Discrete Isocontouring Algorithm." *arXiv*, 2401.14906. [source](https://arxiv.org/abs/2401.14906)

[8] Keeter, M. (2020). "QEF Explainer." mattkeeter.com. [source](https://www.mattkeeter.com/projects/qef/)
