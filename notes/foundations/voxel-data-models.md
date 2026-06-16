<link rel="stylesheet" href="./css/globals.css">

# what a voxel stores

The same grid of cells can be a Minecraft world, a CT scan, a smoke simulation, or
a point cloud — and the only thing that changes is what each cell holds. Before you
write a single line of code, the most consequential decision in any voxel system is
the **payload**: what does one cell actually contain?

That choice is not cosmetic. It quietly determines which meshing algorithms are
even legal, which render path is reachable, and how badly memory explodes as the
grid grows. Pick the wrong payload for what you need to *do* and no amount of
optimization rescues you later.

This page surveys the five main payload families, explains what each one enables
and what it costs, and shows how they connect to the rest of the pipeline.

---

## the coarse model — one choice, many consequences

A voxel grid is a 3-D array. Every cell sits at a fixed position; the grid
doesn't change shape. What can change is the *value* parked in each cell:

- a single bit (full or empty)
- a real number (how full, or how far to the nearest surface)
- a small integer (which material this is)
- a color tuple (what light reaches the eye from this cell)
- a struct combining several of the above

Each option opens some doors and closes others. The sketch below is the mental
model to carry through the rest of this page:

```
payload type      → enables                    → costs
────────────────────────────────────────────────────────────
single bit        → blocky mesh, collision     → nothing useful about surface shape
scalar (density)  → smooth iso-surface         → one float per cell, transfer function needed
scalar (SDF)      → sharp smooth surface       → maintaining signed distance is hard
material id       → rich block worlds          → shape is still blocky without density
color / RGBA      → volume rendering           → 4× storage vs single scalar
multi-attribute   → simulation, engines        → many bytes per cell × many cells
```

A beginner can stop here with a true model: the payload is the one design choice
that locks in — or rules out — everything downstream.

---

## payload type 1 — the occupancy bit

The simplest possible payload: one bit per cell, meaning *occupied* or *empty*.
That is it. No material, no color, no sub-cell geometry.

<em>Occupancy grid</em> is the name for a voxel grid that stores only this
binary state. Elfes introduced the concept formally in 1989 for mobile robot
navigation, where sensors produce noisy range measurements and the robot needs a
compact spatial model it can update incrementally. Each cell holds a probability
estimate rather than a hard bit in the original formulation — but the simplest
game and sim use cases collapse that to a pure boolean.

([local PDF](../papers/elfes-1990-occupancy-grids-stochastic-spatial-representation.pdf) · [arXiv:1304.1098](https://arxiv.org/abs/1304.1098))

**What it enables:**

- Fast collision detection — a cell lookup tells you instantly whether a point
  in space is blocked.
- Blocky meshing — quads are placed on the boundary between occupied and empty
  cells. This is the foundation of [blocky and greedy meshing](../meshing/blocky-and-greedy-meshing.md).
- Cheap broad-phase physics — spatial hashing over the occupied set is
  straightforward.
- Minimal memory — 8 cells per byte (one per bit), or with byte alignment,
  one byte per cell.

**What it cannot do:**

- No sub-cell surface precision. Every surface sits on a voxel face — that is
  where the blocky look comes from.
- No smooth iso-surface extraction. Marching cubes needs a scalar gradient, not
  a binary flag (see [scalar payloads](#payload-type-2--the-scalar-field) below).
- No material distinction. Every occupied cell is identical; you need at least a
  material id to know what kind of thing it is.

**When to reach for it:**

- Collision maps, navmeshes, sensor occupancy models (robotics).
- Games that embrace the blocky aesthetic as a feature, not a limitation.
- Broad-phase culling layers inside larger pipelines.

---

## payload type 2 — the scalar field

Instead of a single bit, each cell holds a real number — a measurement of
*something* at that point in space. The exact meaning of that number depends on
what the grid represents, but two uses dominate in voxel systems: density and
signed distance.

### density

A density value (say, a float from 0 to 1) measures how much matter occupies
the cell. Zero means empty; one means completely full. Values in between represent
partial occupancy — the cell is on or near a surface.

The value Godot Voxel Tools and similar engines use is exactly this: an 8-bit
quantized density where 0 is air and 255 is solid interior, with the surface
inferred somewhere in between. The 8-bit step is ~0.004 in normalized space,
which is fine for terrain; tighter tolerances need 16-bit.

### signed distance

A more powerful scalar: the <em>signed distance</em> at a cell is the distance
from that cell's center to the nearest surface, with a sign to indicate which
side you are on. Negative inside the object, positive outside, zero exactly on
the surface.

Curless and Levoy (SIGGRAPH 1996) used this representation to fuse depth-camera
frames into a coherent 3-D model — each incoming depth frame votes on the signed
distance at each voxel, and the accumulated weighted average converges to a clean
surface. They called the representation a TSDF (Truncated Signed Distance
Function) because the values are clamped to a narrow band around the surface;
cells deep inside or far outside carry no useful information.

([local PDF](../papers/curless-levoy-1996-tsdf-volumetric-range-images.pdf) · [SIGGRAPH 1996 via ACM](https://dl.acm.org/doi/10.1145/237170.237269))

### the iso-surface idea

Both density and SDF share a critical property: they are continuous scalar
fields that *cross zero* (or cross some chosen threshold) at the surface. That
crossing point is the <em>iso-surface</em> — the surface defined by all cells
where the scalar equals a chosen <em>iso-value</em>.

For an SDF, the natural iso-value is 0 (exactly the surface). For density, you
might choose 0.5 (half-full cells sit on the surface). The surface is not tied
to any particular voxel face; it floats through the grid at sub-voxel precision
wherever the scalar crosses the threshold.

**What a scalar field enables:**

- Smooth surface extraction via [marching cubes](../meshing/marching-cubes.md)
  (Lorensen & Cline, 1987) — the algorithm visits each cube of eight neighboring
  voxels, classifies which corners are inside vs outside the iso-surface, and
  triangulates accordingly. Newman & Yi's 2006 survey covers the full algorithm
  family. ([local PDF](../papers/newman-yi-2006-survey-marching-cubes.pdf) · [Computers & Graphics, 2006](https://doi.org/10.1016/j.cag.2006.07.021))
- Dual contouring (Ju, Losasso, Schaefer, Warren — SIGGRAPH 2002) — uses the
  same scalar field but also stores the gradient at edge crossings (Hermite data),
  enabling feature-preserving smooth meshes with sharp creases. Covered in
  [surface nets and dual contouring](../meshing/surface-nets-and-dual-contouring.md).
- Volume rendering — a scalar field doubles as absorption data when each cell's
  value is treated as local density for light transport.

**What it costs:**

- At minimum, one float per cell (4 bytes). A 512³ grid at 4 bytes per cell is
  512 MB before you store anything else.
- Maintaining a valid SDF as the geometry changes is non-trivial — fast SDF
  updates are an active research area.
- A transfer function is needed to map scalar values to visible properties for
  rendering (see [color / RGBA](#payload-type-4--color-and-appearance) below).

**Quantization:** Storing a full 32-bit float per cell is often wasteful. A
signed 8-bit integer (−128 to 127) gives 256 levels of signed distance, which
is enough for most terrain SDFs. Godot Voxel Tools defaults to 16-bit fixed-point
for density (range ±500, step ~0.015). The tradeoff: narrower quantization means
coarser surface precision, but 4–8× memory savings.

---

## payload type 3 — the material id

Games and CAD tools often need to know *what kind* of thing a cell is, not just
whether it is full. A material id (or block id) stores a small integer — an
index into a table of material definitions. The definitions live outside the grid;
the voxel just carries the pointer.

<em>Material grid</em> or *block grid* is the informal name. Minecraft is the
canonical example: each cell stores a block type (stone, dirt, oak wood, etc.)
as a 16-bit id. The geometry is always blocky because there is no scalar field
to interpolate — but the visual variety comes from the per-cell category.

### palette compression

A naive material grid stores the full integer at every cell. A smarter layout
uses a <em>palette</em>: the actual type definitions go in a small lookup table
(the palette), and each voxel stores only a short index into that table. If a
chunk has only 16 distinct block types, each voxel index needs just 4 bits
instead of 16, cutting storage in half. If 2 types, 1 bit. The indices are
bit-packed into a buffer and decoded on access.

The palette exploits the fact that real voxel worlds are *low-entropy*: large
regions are homogeneous (mostly air, mostly stone), so the unique-type count
per chunk is almost always much smaller than the theoretical maximum. See
[dense grids and chunks](../storing/dense-grids-and-chunks.md) for how palette
compression is implemented in chunk storage.

**What it enables:**

- Per-cell identity: different materials can have different physics, sounds,
  render shaders, and simulation rules.
- Efficient chunk storage via palette compression.
- Material-aware meshing: adjacent same-material faces can be merged (greedy
  meshing), reducing triangle counts significantly.

**What it cannot do:**

- No smooth surfaces — you still need a scalar payload for that. Some engines
  pair a material id with a density value (two channels), getting both smooth
  surfaces and per-cell material identity.

---

## payload type 4 — color and appearance

The simplest form of appearance data is an RGBA tuple per cell: red, green,
blue, and alpha (opacity). This makes each voxel its own tiny colored block.
Point-cloud renderers and artistic voxel tools (MagicaVoxel) work this way.

For volume rendering, the payload expands to include two optical properties:

- <em>Emission</em>: how much light the cell radiates on its own (fire, hot
  gas, fluorescence).
- <em>Absorption</em>: how much light the cell blocks as a ray passes through
  it.

Drebin, Carpenter, and Hanrahan (SIGGRAPH 1988) formalized this emission-
absorption optical model: a ray accumulates color emitted along its path while
simultaneously being attenuated by absorption. The discrete approximation is
front-to-back compositing — each sample adds a fraction of color, weighted by
its opacity, and reduces the remaining transparency for later samples.

In practice, the voxel itself often stores a single scalar (density), and a
*transfer function* maps that scalar to emission and absorption at render time.
This decouples the data from the visualization: you can re-color a CT scan by
editing the transfer function without touching the voxel data. See [volume ray
casting](../rendering/volume-ray-casting.md) for how this pipeline works.

Crassin et al.'s GigaVoxels (I3D 2009) and the later voxel cone tracing paper
(EGPGV 2011) extended this: voxels in the octree store pre-filtered color and
opacity at multiple LOD levels, enabling GPU-efficient cone queries that can
approximate soft shadows and indirect lighting. The local PDF is at
[`../papers/crassin-2011-voxel-cone-tracing.pdf`](../papers/crassin-2011-voxel-cone-tracing.pdf)
(from [INRIA HAL](https://inria.hal.science/hal-00650173v1)).

**What color/RGBA enables:**

- Direct volume rendering — ray marching through the grid accumulates color and
  opacity without ever extracting a surface mesh.
- Medical visualization (CT, MRI): density → transfer function → color.
- VFX clouds, smoke, fire: density + temperature → emission + absorption.

**What it costs:**

- 4 bytes per cell for RGBA (vs 1 for occupancy, 1–4 for SDF). A 512³ RGBA
  grid is 2 GB.
- Transfer functions add a design step that raw occupancy or meshing pipelines
  avoid.

---

## payload type 5 — multi-attribute structs

Simulation engines and game engines often need several values per cell at once.
A fluid sim might need:

- density (how much fluid)
- velocity (direction and speed)
- temperature (for thermal dynamics)
- pressure

A game engine might need material id + density (for smooth terrain with typed
blocks) + a baked light level + a custom flag byte.

These are packed into a per-voxel struct. Ken Museth's VDB data structure (ACM
TOG 2013) is the canonical engineering solution for multi-attribute sparse
volumes: it represents each attribute as a separate tree channel so they can be
accessed independently, iterated efficiently, and processed on GPU. VDB stores
any arithmetic or vector type per leaf — the payload schema is defined at
instantiation time.

([local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf) · [ACM TOG 2013](https://dl.acm.org/doi/10.1145/2487228.2487235))

**What multi-attribute enables:**

- Full simulation state in the grid: advect velocity, update temperature, re-compute density all in one pass.
- Simultaneous surface extraction (from density) and material-aware shading.
- Per-cell baked data (light, ambient occlusion, custom gameplay flags).

**What it costs:**

- Every added attribute multiplies total storage. If a single-float density grid
  is 512 MB, adding velocity (3 floats) and temperature (1 float) quintuples it
  to 2.5 GB before compression.
- Cache behavior degrades as structs grow: reading density for a meshing pass
  also loads temperature and velocity into cache even if they are not needed.

---

## how payload size multiplies with cell count

This is the central pressure behind all of [the storage problem](../storing/the-storage-problem.md):

```
total bytes = cells × bytes_per_cell
            = (X × Y × Z) × payload_size
```

A 256³ grid (about 16 million cells):

| payload                  | bytes/cell | total      |
|--------------------------|-----------|------------|
| occupancy (1 bit)        | 0.125     | 2 MB       |
| 8-bit density            | 1         | 16 MB      |
| 32-bit SDF float         | 4         | 64 MB      |
| RGBA (4 bytes)           | 4         | 64 MB      |
| density + material + SDF | 6         | 96 MB      |
| VFX struct (density + velocity + temp) | 16 | 256 MB |

Double the grid size in each dimension: memory goes up 8×. Richer payloads
explode storage faster because the multiplier is always the full cell count.
This is why sparse representations matter so much — occupying memory only for
non-empty cells is covered in [the storage problem](../storing/the-storage-problem.md)
and [dense grids and chunks](../storing/dense-grids-and-chunks.md).

---

## the payload locks in the algorithm path

The payload type does not just affect storage — it determines which algorithms
you can even call:

```
occupancy (bit)
  → blocky meshing, greedy quad merging
  → collision, navmesh queries
  → NOT: marching cubes, volume rendering

scalar field (density / SDF)
  → marching cubes, dual contouring (smooth mesh)
  → sphere tracing / ray-marching SDFs
  → with a transfer function: volume rendering too

material id
  → blocky meshing with per-block material assignment
  → NOT: smooth surface (needs a scalar channel alongside)

color / RGBA / density
  → direct volume rendering (ray marching, compositing)
  → NOT: a clean polygonal mesh (use scalar path for that)

multi-attribute struct
  → simulation update passes (velocity, temperature, pressure)
  → whichever mesh/render path the density channel supports
```

The practical implication: if you decide mid-project that you need smooth
terrain, and you built your grid with occupancy bits, you are replacing the
payload schema — not patching the mesher.

---

## specifics — the fine-grained choices

### air as the implicit default

Every voxel grid has a concept of "empty" — the value that unoccupied cells
hold. For occupancy, that is 0. For a density grid, it is 0.0 (or −1.0 for an
SDF). For a material grid, it is a reserved "air" id (usually 0).

In sparse representations, empty cells are *not stored at all* — the implicit
value is assumed wherever no explicit entry exists. This makes the choice of
"what counts as empty" load-bearing: a cell that stores the air material id
explicitly wastes memory in a sparse structure.

### quantization

Full 32-bit floats per cell are rarely necessary:

- SDF in a terrain context: 8-bit signed integer gives 256 levels over the
  truncation band — usually enough.
- Density for smooth meshing: 8- or 16-bit unsigned is the standard. The Godot
  Voxel Tools documentation uses 16-bit by default (range ±500, step ~0.015).
- Color: 8 bits per channel (RGBA32) is standard; HDR channels need 16-bit
  (RGBA64) or packed 10-10-10-2 formats.

The rule: quantize to the minimum resolution the downstream algorithm tolerates,
measure the error, and only widen if artifacts appear.

### interleaved vs per-attribute layout

When a voxel carries multiple attributes, two memory layouts compete:

**Array of Structs (AoS)** — interleaved. All attributes for voxel 0, then all
for voxel 1, and so on. Efficient when you need every attribute of one voxel at
once (e.g., building a mesh vertex that needs density + material + color).

**Struct of Arrays (SoA)** — planar. All densities, then all materials, then all
colors. Efficient when you process one attribute across many voxels (e.g., a
meshing pass that reads only density, or a lighting pass that reads only color).
GPU shaders almost always benefit from SoA because SIMD lanes can read adjacent
density values in parallel without stride gaps. SoA can run ~30% faster on CPU
and 10× faster on GPU for attribute-specific passes.

VDB takes the SoA approach at the tree level: each attribute lives in a separate
tree channel, so iterating density never pays for velocity. The tradeoff comes
in voxels that need per-cell access across attributes: the indirection adds a
small overhead. Forward reference: [memory layout](../storing/dense-grids-and-chunks.md)
covers these tradeoffs in depth.

---

## summary — choose the payload that matches the job

| goal                              | reach for                          |
|-----------------------------------|------------------------------------|
| collision, navmesh, simple sim    | occupancy bit                      |
| blocky game world with identity   | material id + palette compression  |
| smooth terrain or organic shapes  | density or SDF scalar              |
| feature-sharp smooth surfaces     | SDF + Hermite data (dual contouring) |
| clouds, smoke, CT visualization   | density scalar + transfer function |
| fully colored point-cloud world   | RGBA per cell                      |
| fluid sim, multi-physics          | multi-attribute struct (VDB style) |

The rest of the site shows what each path looks like in practice:
[what is a voxel](./what-is-a-voxel.md) · [the voxel grid](./the-voxel-grid.md) ·
[the storage problem](../storing/the-storage-problem.md) ·
[marching cubes](../meshing/marching-cubes.md) ·
[blocky and greedy meshing](../meshing/blocky-and-greedy-meshing.md) ·
[volume ray casting](../rendering/volume-ray-casting.md)

## references

- Elfes, A. (1989). "Using Occupancy Grids for Mobile Robot Perception and Navigation." *IEEE Computer*, 22(6), 46–57. [local PDF](../papers/elfes-1990-occupancy-grids-stochastic-spatial-representation.pdf) · [arXiv:1304.1098](https://arxiv.org/abs/1304.1098)
- Curless, B. and Levoy, M. (1996). "A Volumetric Method for Building Complex Models from Range Images." *SIGGRAPH '96*. [local PDF](../papers/curless-levoy-1996-tsdf-volumetric-range-images.pdf) · [source](https://dl.acm.org/doi/10.1145/237170.237269)
- Newman, T. S. and Yi, H. (2006). "A Survey of the Marching Cubes Algorithm." *Computers & Graphics*, 30(5), 854–879. [local PDF](../papers/newman-yi-2006-survey-marching-cubes.pdf) · [source](https://doi.org/10.1016/j.cag.2006.07.021)
- Crassin, C., Neyret, F., Sainz, M., Green, S., and Eisemann, E. (2011). "Interactive Indirect Illumination Using Voxel Cone Tracing." *Computer Graphics Forum*, 30(7). [local PDF](../papers/crassin-2011-voxel-cone-tracing.pdf) · [source](https://inria.hal.science/hal-00650173v1)
- Museth, K. (2013). "VDB: High-Resolution Sparse Volumes with Dynamic Topology." *ACM Transactions on Graphics*, 32(3). [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf) · [source](https://dl.acm.org/doi/10.1145/2487228.2487235)
