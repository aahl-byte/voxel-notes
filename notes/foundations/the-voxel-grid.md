<link rel="stylesheet" href="./css/globals.css">

# the voxel grid

To put voxels to work, every piece of code touching the data — renderer, physics
solver, meshing algorithm — has to agree on two things: which cell in the array
corresponds to a given point in the world, and which cells are adjacent to any
given cell. That shared agreement is the grid. Everything else — data models,
compression, rendering — rests on it.

---

## the coarse model

Take a rectangular box in 3D space. Slice it into equal-sized cells along each
axis — say 256 cells wide, 256 deep, 256 tall. You now have a 3D array. Each cell
holds one value. The position of that cell in the real world is determined
entirely by where the box starts (the <em>origin</em>), how many cells fit along
each axis (<em>resolution</em>), and how large each cell is in world units
(<em>voxel size</em>).

That is the voxel grid. Everything else on this page is precision added to that
picture.

- the **origin** is the world-space position of cell (0, 0, 0) — it anchors the
  whole grid in space.
- the **resolution** (W × H × D cells) controls how many samples you get.
- the **voxel size** (or *spacing*) controls how much of the world each cell
  covers.
- the **bounding extent** is just `origin + resolution * voxel_size` — you don't
  need to store it separately; it follows from the other three.

With origin, resolution, and voxel size in hand, you can answer every spatial
question about the grid using arithmetic alone.

---

## moving parts

### from continuous world space to discrete grid coordinates

The data you want to voxelize lives in continuous world space — positions in
millimetres, metres, or game units. The grid lives in discrete integer index
space: `(i, j, k)` where `i ∈ [0, W)`, `j ∈ [0, H)`, `k ∈ [0, D)`. Getting
between them takes two operations: a translation (to line the origins up) and a
scale (to convert world units to cell units).

**world → grid (continuous):**

```
p_grid = (p_world - origin) / voxel_size
```

This gives a floating-point grid coordinate. The integer cell index is just the
floor of that:

```
(i, j, k) = floor(p_grid)
```

**grid → world (the inverse):**

```
p_world = origin + p_grid * voxel_size
```

That is the entire transform for a uniform axis-aligned grid. No matrices needed
unless the grid can be rotated. OpenVDB generalises this to a full affine map
(scale, rotation, shear, translation) and calls the mapping a *Transform*,
exposing `indexToWorld` and `worldToIndex` methods — but under the hood, the
uniform axis-aligned case reduces to exactly the translate-and-scale above
[Museth 2013, [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf),
[original](https://dl.acm.org/doi/10.1145/2487228.2487235)].

### cell center vs cell corner

There are two conventions for where the value stored at index `(i, j, k)`
*lives* in world space.

**Cell-centered** — the value sits at the center of the cell:

```
p_center = origin + (i + 0.5) * voxel_size
```

**Cell-corner (vertex-centered)** — the value sits at the corner of the cell,
which is also a lattice vertex:

```
p_corner = origin + i * voxel_size
```

The two choices disagree on where the data "is" by half a voxel. This matters
the moment you compare grids or sample between them.

- **Medical imaging** (VTK, ITK, NiBabel, all standard DICOM toolkits) uses
  cell-centered by default — the stored value describes the tissue at the center
  of the scanned volume element.
- **2D rasters and some game engines** favor cell-corner because grid vertices
  align with integer screen coordinates.
- **OpenVDB** documents both, names them explicitly, and uses cell-centered for
  its standard volume workflows.

When building a system that mixes data sources, check the convention first. A
half-voxel offset that goes unnoticed compounds into visible misregistration.

### the world↔grid transform when the grid can be rotated

A uniform axis-aligned grid needs only origin and voxel size. A grid that can
also be rotated (or sheared) needs a full 4×4 affine matrix:

```
p_world = M * [i, j, k, 1]^T
```

where `M` encodes the rotation, scale, and translation all at once. The inverse
`M⁻¹` converts world → index. DICOM CT volumes always carry this matrix
implicitly: `ImagePositionPatient` (tag `0020,0032`) sets the origin;
`ImageOrientationPatient` (tag `0020,0037`) gives the row and column direction
vectors; `PixelSpacing` (tag `0028,0030`) gives x/y voxel size; and slice
spacing (derivable from consecutive `ImagePositionPatient` values) gives z size.
Assembling these four fields produces the exact 4×4 affine that maps pixel index
to patient coordinates in millimetres.

See [scanned and volume data](../generating/scanned-and-volume-data.md) for how
this is handled end-to-end for medical CT acquisitions.

---

## sampling between cells

Once you have a world point mapped to continuous grid coordinates, you need to
read a value. You have a continuous position; the data only exists at integer
indices. You must choose how to blend the surrounding stored values.

### nearest neighbor

Return the value of whichever cell contains the point — i.e., round to the
nearest integer index:

```
(i, j, k) = round(p_grid)     # or floor(p_grid + 0.5)
value = grid[i][j][k]
```

- **cost:** one memory lookup.
- **result:** correct at cell centers; produces a blocky, stairstepped field
  everywhere else.
- **when to use:** binary occupancy grids, voxel art, any case where you
  deliberately want sharp cell boundaries.

### trilinear interpolation

Weight the eight cells surrounding the sample point by proximity. The fractional
part of the grid coordinate (how far the sample sits from the nearest lower
corner) gives the weights:

```
tx = p_grid.x - floor(p_grid.x)    # fractional offset, 0..1
ty = p_grid.y - floor(p_grid.y)
tz = p_grid.z - floor(p_grid.z)

value = (1-tx)(1-ty)(1-tz) * c000
      +    tx (1-ty)(1-tz) * c100
      + (1-tx)   ty (1-tz) * c010
      +    tx    ty (1-tz) * c110
      + (1-tx)(1-ty)   tz  * c001
      +    tx (1-ty)   tz  * c101
      + (1-tx)   ty    tz  * c011
      +    tx    ty    tz  * c111
```

where `c000` is `grid[i][j][k]`, `c100` is `grid[i+1][j][k]`, and so on.

If the grid is cell-centered, apply a −0.5 offset before extracting the
fractional part so that a sample exactly at a cell center returns that cell's
value cleanly — without the offset, the weights straddle two cells incorrectly.

- **cost:** eight memory lookups plus arithmetic.
- **result:** smooth, continuous field; C⁰ continuous (smooth values,
  discontinuous derivatives at cell boundaries).
- **when to use:** density fields, SDF volumes, medical imaging, any continuous
  scalar or vector field you need to read at arbitrary positions.

Trilinear is the default choice for volume rendering and physics. Tricubic (64
neighbors, C¹ continuous) is available when smoothness of derivatives matters,
but is rarely worth the cost in real-time contexts.

---

## linear addressing: flattening (i, j, k) into a 1D index

A 3D array lives in linear memory. The formula that maps a grid coordinate to a
memory position is called the <em>linear index</em>:

```
index = x + y * W + z * W * H
```

where `W` is width (cells along x) and `H` is height (cells along y). This is
**x-major** order: as `x` increments by 1, the memory address increments by 1.
Walking along x is a sequential memory access — cache-friendly. Walking along z
steps by `W * H` elements between each access — potentially cache-hostile on
large grids.

This ordering is the standard C-language convention applied to 3D arrays (`a[z][y][x]`
in C, where the rightmost subscript varies fastest). Fortran and some scientific
libraries use the reverse (z varies fastest), which would give `index = z + y*D + x*D*H`.
The formula itself does not care which you call "x" and which you call "z" — what
matters is that you pick one convention and apply it everywhere.

**Neighbor access is just arithmetic on the index:**

| neighbor | index offset |
|---|---|
| ±x | ±1 |
| ±y | ±W |
| ±z | ±W*H |

This means that once you hold an index, you can reach any of the six face
neighbors with a single add or subtract — no multiplication. The 18- and
26-connected neighbors (edge- and corner-adjacent cells) add combinations of
those three offsets.

The memory layout the grid uses — which axis varies fastest — also determines
which neighbor access patterns hit the cache and which don't. See
[memory layout and Morton curves](../optimization/memory-layout-and-morton.md)
for how Morton (Z-order) encoding interleaves the bits of `(x, y, z)` to
produce an index where all six face neighbors are much closer together in memory
than they would be under row-major order.

---

## specifics

### anisotropic spacing

So far we have assumed `voxel_size` is a single number — the same in all three
axes. When each axis has its own spacing, the grid is <em>anisotropic</em>:

```
p_world.x = origin.x + i * spacing_x
p_world.y = origin.y + j * spacing_y
p_world.z = origin.z + k * spacing_z
```

Anisotropic grids are the rule in medical CT, not the exception. In-plane pixel
spacing (`spacing_x`, `spacing_y`) is set by the CT detector pitch and
reconstruction FOV — typically 0.5–0.9 mm. Slice spacing (`spacing_z`) is set
by the table feed and reconstruction protocol — often 0.5–5 mm, sometimes much
larger for scout views. The same box of anatomy can be represented at 0.7 × 0.7
× 3 mm even though it is nominally "isotropic" in the lateral plane.

Code that assumes isotropic spacing will compute wrong distances, wrong gradients,
and wrong surface normals from an anisotropic CT volume. Algorithms that need
isotropy (marching cubes normals, for instance) must either resample to isotropic
spacing first, or weight each axis by its spacing when computing distances.

See [scanned and volume data](../generating/scanned-and-volume-data.md) for the
exact DICOM tags that carry this information.

### uniform vs non-uniform grids

The page so far has covered **uniform** grids — constant spacing within each
axis, same spacing across all axes (or at least per-axis constant). Two
departures are common:

- **Anisotropic uniform** — spacing is constant per axis but differs between
  axes (CT, MRI; described above).
- **Non-uniform / adaptive** — spacing varies *within* an axis. Octrees, sparse
  grids, and VDB structures all achieve this by subdividing space differently in
  different regions. The world↔grid transform then becomes non-trivial because
  a single scale factor no longer covers the whole grid.

The grids described on this page are uniform and axis-aligned. Non-uniform and
adaptive structures are covered in [voxel data models](./voxel-data-models.md)
and [the storage problem](../storing/the-storage-problem.md).

### resolution, voxel size, and memory cost

Resolution and voxel size are linked by the size of the world region you want to
cover:

```
resolution_axis = ceil(world_extent_axis / voxel_size)
```

Halving the voxel size doubles the resolution on each axis — and multiplies the
cell count (and memory cost) by 8. This is the central tension of voxel data:
finer resolution is always more expensive in all three dimensions simultaneously.
A dense 1024³ grid at 4 bytes per cell costs 4 GB. This is why sparse
representations (covered in [dense grids and chunks](../storing/dense-grids-and-chunks.md))
exist: most grids are nearly empty and can skip allocating storage for air.

---

## what this page rests on

- [what is a voxel](./what-is-a-voxel.md) — the cell itself; what a single
  value in the grid represents.
- [voxel data models](./voxel-data-models.md) — the range of structures
  built on top of a grid: dense arrays, chunked arrays, octrees, hash maps, VDB.

## where this leads

- [the storage problem](../storing/the-storage-problem.md) — why a naive dense
  grid is often unaffordable and what to do about it.
- [dense grids and chunks](../storing/dense-grids-and-chunks.md) — the simplest
  concrete storage structure that implements the grid described here.
- [memory layout and Morton curves](../optimization/memory-layout-and-morton.md)
  — how index ordering affects cache performance, and how Morton encoding
  improves on row-major for 3D grids.
- [scanned and volume data](../generating/scanned-and-volume-data.md) — how
  real CT/MRI acquisitions populate the grid, including the full DICOM affine
  and anisotropic spacing in practice.

## references

- Museth, K. (2013). "VDB: High-Resolution Sparse Volumes with Dynamic Topology." *ACM Transactions on Graphics*, 32(3), Article 27. (The OpenVDB transform model — `indexToWorld` / `worldToIndex` — generalises the grid mapping described here to a full affine map.) [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf) · [source](https://dl.acm.org/doi/10.1145/2487228.2487235)
