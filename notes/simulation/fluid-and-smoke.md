<link rel="stylesheet" href="./css/globals.css">

# fluid and smoke

Smoke curling around a character's shoulder as they step through a portal. A fireball that expands, mushrooms, then pulls inward on itself. Water that sloshes in a bucket, sheets off a cliff face, and splashes into droplets — each one catching the light differently. These are the effects that fluid simulation on voxel grids makes possible in games and film VFX.

The destination is a convincing, physically-grounded fluid that reacts to the world. Getting there means evolving a velocity field stored across a grid — tracking where the fluid is going — and repeatedly answering two questions each frame: where does everything end up after it moves, and how do we make sure the fluid neither compresses nor expands? Every grid-based fluid solver, from a 100-line game prototype to a DreamWorks production tool, is built from those two operations.

---

## the grid as discretization — an Eulerian solver

The approach that makes grid-based fluid practical starts by dividing the simulation volume into a regular 3D grid of cells — exactly the same voxel grid described in [voxels as a simulation grid](./voxels-as-a-simulation-grid.md). Instead of tracking individual fluid parcels moving through space (the Lagrangian view), the solver asks, at each fixed grid location: what is the velocity here right now? Then it evolves those stored values step by step. This fixed-grid perspective is called <em>Eulerian</em>.

Each cell stores:

- **velocity** — a vector (u, v, w) describing how fast fluid is moving through that cell. In practice these are stored on cell faces rather than cell centers (the staggered <em>MAC grid</em>, Marker-And-Cell), so the x-velocity lives on left/right faces, y-velocity on top/bottom faces, z-velocity on front/back faces. This placement makes the incompressibility condition easy to check.
- **density** (for smoke) — how much smoky material is here, used at render time to compute opacity.
- **temperature** (for smoke/fire) — drives buoyancy: hot gas rises, cool gas sinks.
- **pressure** — a scalar that the solver computes and discards each step; it is the corrective field that keeps the fluid from bunching up or thinning out.

The equations that govern how these fields evolve are the <em>Navier–Stokes equations</em> — the standard model of fluid motion. On a grid, solving them exactly at every timestep is prohibitively expensive. What made grid-based solvers practical for graphics was learning how to split the solve into two tractable pieces.

---

## the two operations that recur every frame

Every timestep of a grid-based fluid solver reduces to two phases, performed in order. Understanding what each one is *for* is the key to understanding the whole solver.

### advection — moving quantities along the flow

The fluid carries things with it. Whatever density, temperature, or velocity exists at a cell this frame gets transported downstream by next frame. The operation that does this transport is <em>advection</em>.

The naive approach — push quantities forward along the velocity field — is numerically unstable: small errors grow and the simulation explodes. Jos Stam's 1999 paper "Stable Fluids" [1] introduced the fix that the field now uses. Instead of pushing forward, trace *backward*: for each grid cell, follow the velocity field in reverse for one timestep and ask "where would fluid at this cell have come from?" Sample whatever quantity exists at that source location and bring it forward to the current cell. This <em>semi-Lagrangian backtrace</em> is unconditionally stable — no matter how large the timestep, the simulation cannot blow up — because the source point is always within the existing grid, so interpolation always produces a bounded value.

The cost is that the backtraced sample is usually between grid points and must be interpolated. That interpolation blurs the values slightly, causing what is called <em>numerical diffusion</em>: over many steps, fine details — thin wisps of smoke, tight vortices — get smeared out. Stable Fluids' stability guarantee came with this tradeoff, and much of the research since has been about recovering the lost detail.

### pressure projection — enforcing incompressibility

After advection, the velocity field will generally have picked up divergence: some cells will look like they are collecting velocity from their neighbors (compression) and others like they are losing it (expansion). Real water and air are, to a good approximation, incompressible — a parcel of fluid cannot shrink or expand. Enforcing this is the job of <em>pressure projection</em>.

The solver computes a pressure field across the grid by solving a linear system (the Poisson equation for pressure). This is the computationally expensive step — thousands to millions of unknowns, one per active cell. Iterative solvers (Jacobi, conjugate gradient, multigrid) work through the system until the pressure values are consistent. Once the pressure is known, its gradient is subtracted from the velocity field: each cell's velocity is nudged toward its neighbors so that as much fluid leaves each cell as enters it. The result is a <em>divergence-free</em> velocity field — one in which the fluid is neither created nor destroyed anywhere.

These two operations, advect then project, cycle every frame. Stam's contribution was making them both stable enough to run at interactive rates — "Stable Fluids" earned its name because it was the first approach that could take arbitrarily large timesteps without the simulation diverging [1]. His GDC 2003 follow-up "Real-Time Fluid Dynamics for Games" [2] compressed the same ideas into roughly 100 lines of self-contained C, making them accessible to any game developer.

---

## where pure grids struggle — and the hybrid fix

The semi-Lagrangian backtrace is stable, but its interpolation accumulates. Run a pure Eulerian solver long enough and smoke loses its tendrils; water looks like it is moving through syrup even when viscosity is set to zero. This is numerical diffusion, and it is the main limitation of the grid-only approach.

### FLIP/PIC — particles carry detail, the grid solves pressure

The fix is a hybrid: let particles carry the fine-grained velocity detail that the grid smears away, while still using the grid for the expensive pressure solve that enforces incompressibility. The result of combining these two perspectives is called <em>FLIP/PIC</em> (Fluid Implicit Particle / Particle In Cell).

It works in three transfers each step:

1. **Particle → grid:** scatter particle velocities onto nearby grid cells by weighted average.
2. **Grid solve:** advect grid quantities (if needed), apply forces, run the pressure projection to make the velocity divergence-free.
3. **Grid → particle:** instead of replacing particle velocities with the grid values (which would reintroduce diffusion — that is the pure PIC approach), compute only the *change* in grid velocity during the solve and add that delta back to each particle. The particle keeps its accumulated velocity history; the grid only contributes corrections.

This delta-transfer is what makes it FLIP rather than PIC. Because the absolute velocity stays on the particle and only increments come from the grid, the fine-scale swirling motion that the grid would have interpolated away is preserved. The tradeoff is that pure FLIP can be slightly noisy; in practice a weighted blend — mostly FLIP, a small fraction PIC — gives both low diffusion and stable behavior [3].

Zhu and Bridson introduced FLIP to computer graphics fluid simulation in their SIGGRAPH 2005 paper, originally to simulate granular materials like sand [3]. The same hybrid is now the foundation of essentially every production liquid solver.

### the liquid surface — level sets

FLIP/PIC handles the interior of a liquid, but to render water you also need to know exactly where its surface is. A voxel grid is not a natural fit for tracking a moving surface at sub-voxel precision.

The standard solution: store a scalar value at every grid cell that tells you how far that point is from the nearest surface, and which side of the surface it is on. Negative inside the liquid, positive outside, zero exactly on the surface. This is a <em>level set</em>, and it is a direct instance of the signed distance fields covered in [SDF and CSG modeling](../generating/sdf-and-csg-modeling.md).

Each timestep, the level set is advected along with the velocity field (a separate advection pass). The zero crossing — the thin band of cells where the value changes sign — defines the liquid surface. Because it is stored on the voxel grid, topology changes (a droplet detaching from a stream, two blobs merging) happen automatically: cells switch sign as the surface passes through them, with no explicit bookkeeping required. Surface meshes can then be extracted from the level set with marching cubes if needed for rendering.

---

## smoke and fire — extra fields, extra forces

Liquid simulation is mostly about the velocity field and the surface. Smoke and fire simulation adds scalar fields for temperature and density, and those fields drive forces that create the characteristic visual behavior.

### buoyancy

Hot gas rises. Cold gas sinks. This buoyancy force is applied during the external forces step, between advection and projection. The velocity at each cell gets a vertical nudge proportional to the local temperature above (or below) ambient. A simple linear model — upward force = α × density − β × (temperature − ambient) — is enough to produce convincing plumes. The constants α and β control how strongly density weighs the smoke down and how strongly temperature lifts it up.

### vorticity confinement

Numerical diffusion from semi-Lagrangian advection does not just blur density — it also damps the rotational structures in the velocity field. Real smoke has tight, chaotic vortices; on a grid, those vortices decay to smooth gradients within a few steps. Fedkiw, Stam, and Jensen's 2001 smoke paper [4] introduced <em>vorticity confinement</em> as the fix.

The technique works in three sub-steps, applied as another force before projection:

1. Compute the vorticity (curl of the velocity field) at each cell — this measures how much the fluid is spinning locally.
2. Compute a normalized vector that points from low-vorticity regions toward high-vorticity regions.
3. Apply a force along that vector, scaled by vorticity magnitude, that amplifies existing spin rather than damping it.

The effect is that small-scale rotational detail lost to numerical diffusion gets re-injected each frame. The strength parameter ε controls how aggressive the confinement is; too high and the smoke looks artificially noisy, too low and you are back to smeared-out columns.

Fire simulation extends smoke further: a temperature field drives both buoyancy and a reaction term that sources new hot gas (the flame) and converts it to rising cooled combustion products. Density is split into fuel and soot. The visual character of a fireball — expansion, rolling, the afterburn of hot gas rising after the main combustion — all emerge from these coupled fields.

---

## scaling up — sparse grids and GPU solvers

A 256³ grid of velocity vectors at 4 bytes per component costs around 192 MB. A 512³ grid costs 1.5 GB. Worse, most of those cells are empty air that the solver does not need to touch. Two techniques make production-scale fluid simulation practical.

### sparse storage — VDB

Rather than allocating a dense 3D array, production sims allocate only the cells near the active fluid — the region that actually has density or velocity above a threshold. The data structure used for this is the hierarchical B+tree–style grid introduced by Museth in 2013 [5] and open-sourced as <em>OpenVDB</em>. VDB stores only active voxels in a hierarchy of tiles and leaves, with O(1) average access time and efficient streaming. Houdini's fluid solvers use VDB internally; the same grid that stores the simulation is the one exported as a `.vdb` file and ingested by renderers. The storage page covers VDB's architecture in detail — [OpenVDB and NanoVDB](../storing/openvdb-and-nanovdb.md).

For very large sims, VDB can reduce memory use by an order of magnitude or more: a smoke plume that occupies 5% of its bounding volume needs only 5% of the dense-grid memory.

### GPU acceleration

The advection and force steps are embarrassingly parallel — each cell is independent — and map directly onto GPU compute shaders or CUDA kernels. The pressure projection is less parallel (it is a global linear system) but still benefits substantially from GPU execution using iterative solvers (conjugate gradient, multigrid) that process the system in many small passes. A GPU-based fluid solver working on a 128³ grid can run in real time; film-quality sims on multi-hundred-voxel grids use GPU clusters or high-end workstations. The optimization trade-offs — memory bandwidth, sparse work scheduling, tiled dispatch — are covered in [GPU voxel techniques](../optimization/gpu-voxel-techniques.md).

---

## from sim to renderer

The output of a fluid simulation is a set of voxel grids — velocity, density, temperature, level set — typically snapshotted at each frame and saved as VDB files. These files are exactly what a volume renderer ingests. The renderer treats the density field as an extinction coefficient, integrating light extinction along each ray through the volume. That rendering path — and how the sparse VDB structure makes it fast — is explored in [VDB in VFX](../applications/vdb-in-vfx.md).

The connection from sim to render is direct: the same voxel abstraction that made the solver tractable is also the one the renderer expects. No conversion step, no loss of detail — the data flows straight through.

---

## when to reach for each technique

| goal | approach | why |
|---|---|---|
| smoke, gas, explosions | Eulerian grid + vorticity confinement | divergence-free velocity, buoyancy, density field map directly to render |
| splashing water, oceans | FLIP/PIC + level set | low numerical diffusion preserves fine spray; level set gives clean surface |
| game real-time smoke | Eulerian grid on GPU, coarse resolution | stable fluids is unconditionally stable even at large timesteps |
| film VFX fluid | FLIP on VDB, GPU pressure solve | sparse storage handles large domains; FLIP preserves detail at any scale |
| interactive sculpting | level set only, no velocity | SDF edits are fast and the surface is always clean |

The key choice is always the same: do you need to track a surface (level set, FLIP), or do you need volumetric density without a defined boundary (pure Eulerian)? Smoke is the latter. Water is the former.

---

## references

[1] Stam, J. (1999). "Stable Fluids." *Proceedings of SIGGRAPH 1999* (ACM SIGGRAPH), pp. 121–128. DOI: 10.1145/311535.311548. [local PDF](../papers/stam-1999-stable-fluids.pdf) · [source](https://www.josstam.com/publications)

[2] Stam, J. (2003). "Real-Time Fluid Dynamics for Games." *Proceedings of the Game Developers Conference (GDC 2003)*. [local PDF](../papers/stam-2003-realtime-fluid-dynamics-games.pdf) · [source](https://www.josstam.com/publications)

[3] Zhu, Y. and Bridson, R. (2005). "Animating Sand as a Fluid." *ACM Transactions on Graphics (Proc. SIGGRAPH 2005)*, 24(3), pp. 965–972. DOI: 10.1145/1073204.1073298. [local PDF](../papers/zhu-bridson-2005-animating-sand-as-fluid-flip.pdf) · [source](https://history.siggraph.org/learning/animating-sand-as-a-fluid-by-zhu-and-bridson/)

[4] Fedkiw, R., Stam, J., and Jensen, H.W. (2001). "Visual Simulation of Smoke." *Proceedings of SIGGRAPH 2001* (ACM), pp. 15–22. DOI: 10.1145/383259.383260. [local PDF](../papers/fedkiw-stam-jensen-2001-visual-simulation-smoke.pdf) · [source](https://web.stanford.edu/class/cs237d/smoke.pdf)

[5] Museth, K. (2013). "VDB: High-Resolution Sparse Volumes with Dynamic Topology." *ACM Transactions on Graphics*, 32(3), Article 27. DOI: 10.1145/2487228.2487235. [local PDF](../papers/museth-2013-vdb-high-resolution-sparse-volumes.pdf) · [source](https://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf)
