<link rel="stylesheet" href="./css/globals.css">

# cellular automata

Sand that piles up and avalanches. Water that finds its level and floods caves. Fire that jumps from tree to tree and leaves charred ground behind. Acid that eats through whatever it touches. These are the material behaviors that make games like Noita and The Powder Toy feel alive — and every one of them is produced by the same mechanism: a grid of cells, each with a state, each following a tiny local rule.

That is the outcome this page is about. Not the theory of automata — the emergent material behavior that makes simulated worlds feel physical.

---

## the coarse model — how the grid thinks

Here is the idea in plain language, before any terminology.

You have a grid. Every cell in the grid holds a value — a material type, a state, a density. Every tick, each cell looks at itself and at the cells immediately around it. Based on what it sees, it decides what it will be on the next tick. Apply that decision to every cell simultaneously, and the whole grid has advanced one step.

That is the complete mechanism. The cell's state, the set of cells it examines (its <em>neighborhood</em>), and the function that maps state-plus-neighbors to a next state (the <em>rule</em>) — these three things together constitute a <em>cellular automaton</em>. The term is just the name for what you just read.

The reason this produces interesting behavior is that the rules are simple, but they apply everywhere at once, across thousands or millions of cells. Local interactions accumulate into global patterns nobody explicitly programmed.

Conway's Game of Life is the canonical example to understand before anything else. Every cell is either alive (1) or dead (0). Each tick, a cell counts its eight immediate neighbors. If a live cell has two or three live neighbors it survives; otherwise it dies. If a dead cell has exactly three live neighbors it becomes alive; otherwise it stays dead. Four numbers — 2, 3, 3 — govern the entire system. Yet Life produces gliders, oscillators, self-replicating structures. The complexity comes from the grid, not the rule.

Falling-sand and fire simulations use the exact same architecture. They just have more material states and rules tuned to look physical rather than purely mathematical.

---

## the moving parts

### the neighborhood — what a cell can see

A cell can only react to cells it can see. The set of cells it looks at is its neighborhood, and choosing it is the first design decision.

Two standard shapes cover almost all practical uses:

| neighborhood | cells examined | shape |
|---|---|---|
| <em>von Neumann</em> | the 4 direct face-neighbors (up, down, left, right) | plus sign |
| <em>Moore</em> | all 8 surrounding cells (faces + diagonals) | square |

In 3D the same idea extends to 6-face (von Neumann) and 26-neighbor (Moore) neighborhoods.

- **von Neumann** — use when interactions are strictly axis-aligned: water spreading horizontally and vertically, heat diffusing up/down/left/right. Simpler and cheaper.
- **Moore** — use when diagonal interactions matter: sand sliding down-left or down-right past an obstacle, fire leaping diagonally to a neighbor, erosion rounding corners.

Falling-sand simulations almost always use Moore neighborhoods — gravity wants the cell below, but also the two cells diagonally below when the direct path is blocked.

### the rule — what happens at each cell

A rule takes the current state of a cell and the states of its neighbors and returns a new state. In Life the rule is a lookup table indexed by neighbor count. In falling-sand, the rule is a priority list of candidate moves checked in order.

Rules can be:

- **deterministic** — the same inputs always produce the same output (sand falling straight down).
- **probabilistic** — a random factor decides among eligible moves (fire spreading to a Moore neighbor with probability p, not certainty).
- **state-machine** — a cell advances through a sequence of states over time (wood → burning → ember → ash → empty, spending some number of ticks in each).

Most real simulations combine all three: gravity is deterministic, fire spread is probabilistic, and material aging is a state machine.

### applying the rule — the global update

The simplest approach: sweep every cell in the grid, apply its rule, move on. Do this every tick. This is called a <em>synchronous update</em> — all cells advance together in one generation.

In practice, full-grid sweeps have two problems: they are slow on large grids, and the sweep order introduces bias. Both are solved separately (see below).

---

## falling sand concretely

Sand is the easiest case to reason through because gravity gives it a clear goal: get as low as possible.

### the displacement rule

Each tick, a sand cell checks potential destinations in priority order:

1. the cell directly below — if empty, move there.
2. the cell diagonally below-left — if empty, move there.
3. the cell diagonally below-right — if empty, move there.
4. nowhere — stay put.

Water is similar but adds horizontal spreading once it cannot move downward:

1. the cell directly below — if empty (or less dense), move there.
2. diagonally below-left, then diagonally below-right.
3. one step left, one step right (water spreads until it finds a lower exit or fills a basin).

The density comparison is what allows distinct liquids to interact. Oil falls through air but floats on water because the rule checks relative density, not just emptiness. The Powder Toy, for instance, represents many distinct materials with explicit density values and moves a cell only when the destination is less dense than the mover [1].

A short sketch in pseudocode — concept first, code as illustration:

```
# sand update rule (checked each tick for every sand cell)
if cell_below is empty:
    move down
elif cell_below_left is empty:
    move down-left
elif cell_below_right is empty:
    move down-right
# else: stay put (settled)
```

### order-dependence bias and fixes

Here is a real problem. If you sweep the grid from top-left to bottom-right, a sand cell that moves this tick may be re-examined before the tick ends — it moves twice. Water swept left-to-right always spreads rightward, producing an unnatural bias.

These artifacts come from the <em>order-dependence</em> of an in-place, single-buffer update. The fix depends on how much you care:

| approach | what it does | cost |
|---|---|---|
| alternating scan direction | reverse column order on odd ticks | near zero |
| randomized order | shuffle cell visit order each tick | moderate |
| double buffer | write next-state to a second grid, flip at end of tick | 2× memory |
| commit-move list | collect all moves, apply after the full scan | extra pass |

The Noita developers deliberately chose in-place updates (no double buffer) because double-buffering requires updating every cell every tick — you cannot skip settled regions [2]. Instead, they use a dirty flag: once a cell moves in a tick, it is marked and skipped if encountered again in the same sweep. This prevents double-moves without the memory cost of a second grid.

Alternating column direction (left-to-right on even ticks, right-to-left on odd ticks) removes most of the water-bias artifact at essentially no cost [2]. For the highest quality, shuffle the order randomly each tick.

---

## making it cheap — active cells and dirty regions

A world with a million cells where most material has settled is wasteful to sweep in full every tick. The standard solution is to track only the cells that might still be doing something.

### active cells

Every cell that moved or changed state this tick marks itself (and its neighbors, since a neighbor may now be able to move too) as <em>active</em>. Cells that have not changed for some number of ticks go to sleep and are skipped.

- sand at the bottom of a pile: sleeping.
- sand in mid-fall: active.
- sand that just landed: active for a few ticks, then sleeping once settled.

This means a large settled world costs almost nothing to simulate. Only the dynamic regions pay per-tick cost.

### dirty regions (chunk-based tracking)

Instead of tracking individual cells, most implementations track rectangular regions. Noita divides the world into 64×64-pixel chunks. Each chunk maintains a <em>dirty rect</em> — the smallest bounding box that contains all cells that changed last tick [2]. Only the dirty rect of each chunk is processed next tick. Once a chunk's dirty rect collapses to empty, the chunk sleeps.

This is the same approach used in 2D sprite rendering (update only the changed rectangle on screen), applied to simulation. It is described in detail in the [optimization domain](../optimization/gpu-voxel-techniques.md#active-cells).

The gains are substantial: in a typical falling-sand world, the active region is a small fraction of the total grid at any moment.

---

## fire, heat, and acid — more state-transition rules

Sand and water are displacement problems: move a cell to a new location. Fire, heat, and acid are state-transition problems: change what a cell *is* based on what surrounds it.

### fire

A fire simulation typically uses a simple state machine per material:

- **flammable** — can catch fire if a burning neighbor touches it (probabilistic, with a per-material ignition chance).
- **burning** — spreads fire to flammable Moore neighbors each tick; counts down a lifetime.
- **burnt** — consumed, no longer flammable; decays to ash or empty after some ticks.

Gas behavior (smoke, steam) inverts the gravity rule: instead of checking below first, gases check above first and spread upward, then horizontally. Fire in Noita converts water it touches to steam, which then rises — this is just the gas rule activating on a cell that changed state from water [2].

### heat diffusion

Heat is a scalar value per cell. Each tick, the temperature of a cell is updated to a weighted average of itself and its neighbors — a discrete form of diffusion. Cells above a threshold temperature change state (ice → water, wood → burning). This is a continuous-value CA rather than a discrete-state one, but the update structure is identical.

### acid

Acid dissolves materials on contact. The rule: if an acid cell has a Moore neighbor containing a dissolvable material, that neighbor transitions to empty (or to a byproduct), and the acid cell has some probability of being consumed in the reaction. The per-material dissolvability is just a flag the rule checks.

### contrast: CA "water" vs true fluid simulation

This is the most important comparison on this page.

| | cellular automaton water | fluid simulation (Navier-Stokes) |
|---|---|---|
| model | discrete states, local rules | continuous velocity + pressure field |
| physics | approximate (no pressure propagation) | physically accurate |
| water level | fills bottom of container | equalizes correctly with pressure |
| waves | no | yes |
| cost | cheap — rule per cell | expensive — iterative solver |
| best for | games, destructible worlds | film VFX, engineering |

CA water never pushes upward through a sealed pipe. A pressure-based fluid solver (Navier-Stokes) does. If you need water that rises through a submerged tube, fills a U-bend, or produces waves, a CA is the wrong tool. If you need water that floods a cave, splits around an obstacle, and mixes with acid, a CA is perfectly adequate and far cheaper.

The [fluid and smoke page](./fluid-and-smoke.md) covers the full Navier-Stokes approach, the Lattice-Boltzmann method (which sits between CA and NS), and when to reach for each.

---

## gpu implementation — ping-pong buffers

A cellular automaton maps perfectly onto GPU parallelism: every cell's next state is independent of every other cell's next state (given this tick's grid), so all cells can be updated simultaneously.

The standard GPU pattern uses two textures — call them A and B:

1. tick N: read from A, write next-state to B.
2. tick N+1: read from B, write next-state to A.
3. repeat.

This alternating read-write is called a <em>ping-pong buffer</em>. Each cell's rule runs in a fragment shader (or compute shader): sample the neighborhood from the read texture, compute the next state, write the output pixel. Neighborhoods are just texture samples at offset UV coordinates [3].

```glsl
// fragment shader sketch — concept illustration
vec2 px = vec2(1.0 / uWidth, 1.0 / uHeight);
float below    = texture2D(uState, vUV + vec2( 0, -px.y)).r;
float belowL   = texture2D(uState, vUV + vec2(-px.x, -px.y)).r;
float belowR   = texture2D(uState, vUV + vec2( px.x, -px.y)).r;
// apply displacement rule, output new state
```

The GPU ping-pong approach handles millions of cells per tick at interactive framerates, but the double-buffer structure means you cannot do Noita-style single-buffer dirty-flag optimization — you must always write all cells. The practical solution is to use a compute shader with a separate active-cell buffer to skip clearly settled regions, which is covered in [gpu voxel techniques](../optimization/gpu-voxel-techniques.md).

For 3D voxel grids the same pattern uses 3D textures (GL_TEXTURE_3D) with ping-pong between two volume textures. Neighborhood lookups become 3D offsets. See the [runtime editing page](../engines/runtime-editing-and-csg.md) for how this integrates with destructible voxel geometry.

---

## where this fits in the bigger picture

Cellular automata produce the material layer of a simulation grid — the rules that govern what individual voxels do to each other. The grid itself — how to index it, chunk it, and store it efficiently — is covered in [voxels as a simulation grid](./voxels-as-a-simulation-grid.md). Procedural terrain generation often seeds the initial state of a CA world: [procedural terrain](../generating/procedural-terrain.md) explains how noise and erosion rules (themselves a form of CA) produce the starting material distribution that the simulation then animates.

---

## references

[1] The Powder Toy (open-source falling-sand simulator, 2008–). Source: [https://powdertoy.co.uk](https://powdertoy.co.uk). Demonstrates multi-material density comparison, displacement rules, and state-machine material aging.

[2] Purho, P. (2019). "Exploring the Tech and Design of Noita." GDC 2019. [GDC Vault](https://www.gdcvault.com/play/1025695/Exploring-the-Tech-and-Design) · [YouTube](https://www.youtube.com/watch?v=prXuyMCgbTc). Covers the "Falling Everything" engine: dirty-rect chunk tracking (64×64 chunks), checker-pattern threading, single-buffer + dirty-flag update, bottom-up sweep, and per-material CA rules.

[3] Mastripolito, B. (2022). "Cellular Automata in WebGL: Part 1." *Medium*. [https://medium.com/@bpmw/cellular-automata-in-webgl-part-1-df531059f0ab](https://medium.com/@bpmw/cellular-automata-in-webgl-part-1-df531059f0ab). Describes ping-pong double-framebuffer implementation, texture-based state encoding, and fragment-shader neighborhood sampling.

[4] Zucconi, A. (2016). "How to Simulate Cellular Automata with Shaders." *Alan Zucconi*. [https://www.alanzucconi.com/2016/03/16/cellular-automata-with-shaders/](https://www.alanzucconi.com/2016/03/16/cellular-automata-with-shaders/). Lookup-table rule encoding in fragment shaders; avoiding branching with arithmetic.

[5] Macuyiko (2020). "An Exploration of Cellular Automata and Graph Based Game Systems: Part 4." *blog.macuyiko.com*. [https://blog.macuyiko.com/post/2020/an-exploration-of-cellular-automata-and-graph-based-game-systems-part-4.html](https://blog.macuyiko.com/post/2020/an-exploration-of-cellular-automata-and-graph-based-game-systems-part-4.html). Technical analysis of falling-sand update order, order-bias, directional alternation fix, dirty flags, and double-buffering trade-offs.

[6] Gardner, M. (1970). "Mathematical Games: The fantastic combinations of John Conway's new solitaire game 'life'." *Scientific American*, 223(4), 120–123. The original public introduction of Conway's Game of Life as the canonical 2-state cellular automaton example.

[7] Conway's Game of Life — Wikipedia. [https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life). Background on Life, Moore vs von Neumann neighborhoods, and the historical context of cellular automaton theory.

[8] Salami, G. GPU-Falling-Sand-CA — block cellular automata on GPU using Margolus neighborhood. GitHub. [https://github.com/GelamiSalami/GPU-Falling-Sand-CA](https://github.com/GelamiSalami/GPU-Falling-Sand-CA). Demonstrates 2×2 block CA (Margolus) for race-condition-free parallel sand simulation.
