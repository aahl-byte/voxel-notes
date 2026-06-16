<link rel="stylesheet" href="./css/globals.css">

# voxels as a simulation grid

Fire tears through a wooden structure one block at a time, jumping to neighbors that are dry and hot enough to ignite. Water pours into a cave, filling it from the bottom up. A tower of stone crumbles when the blocks below it are destroyed, the collapse rippling upward as each cell checks whether it still has support. Smoke billows and curls as density and velocity in adjacent cells push each other around.

None of these effects require a physics engine that reasons about surfaces, meshes, or rigid bodies in the traditional sense. They emerge directly from a simple idea: every cell in the grid looks at its immediate neighbors, applies a rule, and writes a new value. Repeat that across the entire grid, tick after tick, and complex behavior appears. The grid you already [store and render](../foundations/the-voxel-pipeline.md) is also your simulation substrate — no separate data structure required.

---

## the coarse mental model

A voxel grid is a regular 3D lattice where every cell sits at a known position and has a fixed set of neighbors at constant offsets. That regularity is not just convenient for rendering — it is the exact property that makes the grid a natural place to run simulation.

The mechanism behind every simulation covered in these notes is the same at its core:

- each cell has a **current state** (a material, a density, a velocity, a temperature — whatever the simulation tracks)
- each cell's **next state** is computed from its own current state and the current state of its neighbors
- that computation is swept across the whole grid in one step — a *tick*
- the result replaces the current state, and the tick repeats

This pattern — where each cell's next value is a function of its current <em>neighborhood</em>, swept uniformly over the grid — is called a <em>stencil update</em>. When the neighborhood function is a discrete lookup table of rules rather than a continuous equation, the whole system is called a <em>cellular automaton</em>.

A beginner can stop here and already hold a true model: a voxel simulation is a grid of cells that keep updating from their neighbors.

---

## why the voxel grid is built for this

[What is a voxel](../foundations/what-is-a-voxel.md) explains these properties in full, but they matter enough to name here because they are precisely the reasons simulation on voxel grids is practical:

- **constant-time neighbor access** — the six face-neighbors of the cell at `(x, y, z)` are at `(x±1, y, z)`, `(x, y±1, z)`, and `(x, y, z±1)`. No pointer chasing, no graph traversal, no spatial query. An index offset in an array.
- **in-place update** — every cell holds its own state. Reading a neighbor is reading an array element.
- **locality** — the rule at any given cell depends only on the cells immediately around it. The computation decomposes naturally into independent chunks, which is what lets it run fast on GPU hardware.
- **interiors exist** — unlike a surface mesh, the grid has data everywhere in the volume: inside a rock, in mid-air, deep in a fluid. Simulation can propagate through the interior, not just along an outer shell.

These are not incidental benefits. They are the reason that cellular automata, fluid solvers, heat diffusers, and lighting propagators all land on the voxel grid as their natural home.

---

## the update model in detail

### the stencil — reading then writing

A stencil defines which neighbors a cell reads during an update. For simple heat diffusion, the new temperature at `(x, y, z)` is the average of the six face-neighbors' current temperatures. For fire propagation, the new state is "burning" if the cell is fuel and any neighbor is already burning. The stencil can be as narrow as six face-neighbors or as wide as a 3×3×3 cube of 26 neighbors.

The critical constraint is: **a cell reads from the current state and writes to a separate output**. If a cell read its own already-updated value mid-sweep, cells updated early would influence cells updated later in the same tick. The result would depend on the sweep order — inconsistency rippling through the grid each frame.

The solution is to maintain two copies of the grid, called <em>double buffering</em>: one holding the current state (read-only during a tick) and one accumulating the next state (write-only during a tick). At the end of each tick the roles swap. Reading always comes from the old buffer; writing always goes to the new one. This makes every cell's update independent of every other cell's update within the same tick — the result is order-independent and therefore parallelizable.

```
tick N:   read from buffer A → write results to buffer B
tick N+1: read from buffer B → write results to buffer A
tick N+2: read from buffer A → ...
```

Swapping the buffers costs nothing: you flip a pointer, not data.

### the tick — sweeping the grid

Each tick, every cell in the grid runs its update function. On CPU this is a nested loop over all `(x, y, z)` positions. On GPU it maps directly to a compute shader dispatch: each thread handles one cell, all threads read from the current buffer and write to the next. The regularity of the grid means no thread synchronization is needed within a tick — threads are fully independent, which is exactly what GPU hardware is designed for.

---

## the simulation families

The stencil-update mechanism supports several distinct families of simulation. They share the same update loop; they differ in what state each cell holds and what rule is applied.

### discrete rules — sand, fire, gas

The cell holds a material ID or a small state value. The rule is a lookup table: "if this cell is dry wood and a neighbor is burning, this cell ignites." No continuous math — just table lookups. These are the purest cellular automata. Falling sand, fire spread, gas diffusion, and erosion all belong here. See [cellular automata](./cellular-automata.md) for how these rules are structured and how block-CA approaches make them GPU-friendly.

### propagation — light

The cell holds a light level (often a small integer, like 0–15). The rule: a cell's light level is the maximum neighbor value minus 1. Sweep over the grid and this rule propagates light outward from sources, attenuating with distance, blocked wherever a solid cell stops the spread. No ray casting required. See [light propagation](./light-propagation.md) for the flood-fill variant and how it plugs into the chunk update cycle.

### connectivity and rigid bodies — destruction

The cell holds a material plus metadata about structural support. When cells are destroyed, a connectivity pass identifies which remaining cells can still reach a supported anchor. Unsupported islands become falling rigid bodies. The stencil detects disconnection; the physics of falling is handled separately. See [voxel physics and destruction](./voxel-physics-and-destruction.md).

### continuous fields — fluids, smoke, heat

The cell holds one or more continuous values: velocity, pressure, density, temperature. The update rule comes from a discretized partial differential equation — typically the Navier-Stokes equations for fluids and smoke, or the heat equation for thermal diffusion. The classic grid-based approach is Stam's 1999 stable fluids method [1], which runs unconditionally stable semi-Lagrangian advection on a regular voxel grid. Harris (2004) showed how the same solver maps directly onto GPU programs, reading from one buffer and writing to another — exactly the double-buffer pattern [2]. See [fluid and smoke](./fluid-and-smoke.md) for how the solver steps are decomposed.

| family | cell state | rule type | page |
|---|---|---|---|
| discrete / CA | material ID or state | lookup table | [cellular automata](./cellular-automata.md) |
| propagation | light level | neighbor max minus attenuation | [light propagation](./light-propagation.md) |
| connectivity | material + support flag | flood fill / island detection | [voxel physics and destruction](./voxel-physics-and-destruction.md) |
| continuous fields | velocity, pressure, density | discretized PDE | [fluid and smoke](./fluid-and-smoke.md) |

---

## the render–sim feedback loop

The simulation writes the same cells the renderer reads. A fire simulation turns a cell's material from "wood" to "burning" to "ash" — the mesher or ray-marcher picks up those changes on the next frame and the world looks different. No translation layer is needed: the grid is shared.

This tight coupling has one important practical consequence. The renderer runs every frame at 60 Hz or better; the simulation does not need to. A fluid solver might tick at 30 Hz, a light propagation pass at 20 Hz, a structural integrity check only when a block is destroyed. The sim and render run at different rates; the render thread consumes whatever the sim last committed.

This is also why simulation should run off the main render thread. If the simulation stalls the render, frame rate drops. The standard solution — covered in [the threading and meshing pipeline](../engines/threading-and-meshing-pipeline.md) — is to run simulation as a background compute pass, writing into the shared grid while the render thread reads the grid's last committed state.

On GPU this maps neatly onto double buffering: the simulation dispatches a compute shader that reads from buffer A and writes to buffer B; the renderer samples buffer A in the same frame. The same pattern that makes each tick order-independent also naturally separates the sim write target from the render read source.

---

## references

[1] Stam, J. (1999). "Stable Fluids." *Proceedings of SIGGRAPH 99*, pp. 121–128. DOI: 10.1145/311535.311548. [local PDF](../papers/stam-1999-stable-fluids.pdf) · [source](http://www.dgp.toronto.edu/people/stam/reality/Research/pdf/ns.pdf)

[2] Harris, M. J. (2004). "Fast Fluid Dynamics Simulation on the GPU." In *GPU Gems*, Chapter 38. Addison-Wesley / NVIDIA. DOI: 10.1145/1198555.1198790. [source](https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-38-fast-fluid-dynamics-simulation-gpu)
