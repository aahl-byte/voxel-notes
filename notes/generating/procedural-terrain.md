<link rel="stylesheet" href="./css/globals.css">

# procedural terrain

Minecraft's world is roughly 60 million kilometres across. No Man's Sky contains 18 quintillion planets. Neither game stores that geometry. Instead, every chunk of rock, every cave, every ridge is computed on demand from a handful of mathematical functions that take a position in space and return a number. The world exists as a formula — not as data — and any coordinate regenerates identically every time it is asked for.

This is <em>procedural terrain generation</em>: building a voxel world from pure functions of position rather than from authored or stored geometry. The payoff is effectively infinite variety from almost no storage. The constraint is that the world can only be as rich as the function that defines it. For a survey of all the ways voxel worlds get populated — scanned, authored, and procedural — see [where voxels come from](./where-voxels-come-from.md).

---

## the two shapes of the problem

Before choosing any algorithm, you need to decide what kind of terrain you are modelling. The decision controls everything that follows.

### heightmaps — one height per column

The simplest model samples a 2D function `h(x, z)` to get a surface height, then fills every voxel below that height with solid and everything above with air. You walk over the result and it looks like landscape.

The problem is structural: a <em>heightmap</em> allows exactly one solid/air transition per vertical column. There is no room for a cave beneath the surface, no overhang where rock juts out over empty space, no floating island hovering mid-air. The function `h(x, z)` can only answer "how high is the ground here" — it cannot describe what is underneath.

For many games (rolling hills, simple outdoor worlds) this is enough, and heightmaps are fast to generate and easy to mesh. But the moment you want caves, cliffs with overhangs, or underground structures, the model breaks.

### density fields — one value per point in 3D space

The richer model evaluates a function at every point in the volume: `f(x, y, z) → float`. Every voxel reads the function at its position. If the value is above a threshold — the <em>iso-value</em> — the voxel is solid. If it is below, the voxel is air.

```python
ISO = 0.0

def classify_voxel(x, y, z):
    density = f(x, y, z)   # evaluate the density function
    return "solid" if density > ISO else "air"
```

Now any arrangement of solid and air is expressible. A cave is a region where the density dips below the iso-value. An overhang is solid material that hangs over an air region. A floating island is a pocket of high density suspended in low-density air. The geometry no longer has to be a surface — it is a volume.

The price is computation: you evaluate the function at every voxel, not just one per column. For large worlds this drives chunk-based generation (explored in [chunk management and streaming](../engines/chunk-management-and-streaming.md)).

| | heightmap | 3D density field |
|---|---|---|
| caves / overhangs | no | yes |
| floating islands | no | yes |
| evaluation cost | one call per column | one call per voxel |
| meshing | simple (quads or greedy) | marching cubes / dual contouring |
| best for | outdoor, simple terrain | underground, complex worlds |

Both approaches feed the mesher that turns the voxel classification into renderable geometry — see [marching cubes](../meshing/marching-cubes.md) for how the density field becomes a surface.

---

## the engine of variety — noise

The function `f(x, y, z)` needs to vary smoothly across space so that adjacent voxels produce coherent terrain rather than random static. It also needs to be <em>deterministic</em> — the same position must always return the same value, or chunks loaded on demand will not agree with one another.

The standard tool is a family of <em>noise</em> functions: smooth, pseudo-random functions that vary continuously over space.

### value noise

The simplest variant assigns a random value to each integer lattice point and interpolates smoothly between them. The result is smooth but has a visible "blobby" character — the lattice structure shows through as a subtle grid.

### Perlin noise

In 1985, Ken Perlin described a superior technique in "An Image Synthesizer" [1]. Instead of assigning random values at lattice corners, assign random <em>gradient vectors</em> — unit vectors pointing in random directions. The noise value at any point is computed by interpolating the dot products between those gradients and the offset vectors from each surrounding corner.

The key improvement the 2002 paper [2] made was to the interpolation curve. The original used a cubic `3t² - 2t³`; the revision replaced it with `6t⁵ - 15t⁴ + 10t³` (the "quintic" or "smoothstep" curve), which eliminates visible second-derivative discontinuities at lattice boundaries. In practice: the 2002 version is smoother and was adopted as the standard.

Both versions are <em>gradient noise</em>: the smooth, feature-rich kind of noise most suitable for terrain.

### simplex noise

Perlin's 2002 paper also described <em>simplex noise</em>, which divides space into simplices (tetrahedra in 3D, triangles in 2D) rather than a square/cubic lattice. Stefan Gustavson's 2005 technical note [3] unpacks the algorithm clearly: fewer lattice points contribute to each output value (4 in 3D simplex vs 8 in classic 3D Perlin), the computation is cheaper in higher dimensions, and the isotropy is better — there are no axis-aligned artifacts.

McEwan et al. (2012) [4] demonstrated a GPU-friendly texture-free implementation and measured simplex noise running roughly 20% faster than classic Perlin noise on an NVIDIA GTX580 in 3D.

**When to reach for which:**

- **Value noise** — fast, simple, acceptable for rough prototype or where grid artifacts are harmless.
- **Perlin noise (2002)** — smooth, artifact-reduced, widely supported, good default for terrain.
- **Simplex noise** — lower cost in 3D and above, better isotropy, preferred when generating at multiple scales or on GPU.

---

## layering for detail — fBm

A single noise function produces terrain at one spatial scale: gently rolling hills with no fine ridges, or jagged micro-detail with no large structure. Real terrain has both.

The solution is to sum multiple noise samples at progressively finer scales, each contributing less amplitude than the last. Each layer is an <em>octave</em>. The whole sum is called <em>fractal Brownian motion</em> (fBm), because the resulting signal has the self-similar, scale-invariant character of fractional Brownian motion — the statistical model for natural rough surfaces [5].

The algorithm is a loop:

```python
def fbm(x, y, z, octaves=6, lacunarity=2.0, gain=0.5):
    value = 0.0
    amplitude = 1.0
    frequency = 1.0
    for _ in range(octaves):
        value += amplitude * noise(x * frequency,
                                   y * frequency,
                                   z * frequency)
        frequency *= lacunarity   # zoom in for the next octave
        amplitude *= gain         # contribute less each time
    return value
```

Three parameters control the result:

- **octaves** — how many layers to sum. More octaves add finer detail but cost more compute. Six to eight is typical for terrain.
- **lacunarity** — the frequency multiplier per octave, usually 2.0 (each octave is twice as fine).
- **gain** (also called *persistence*) — the amplitude multiplier per octave, usually 0.5 (each octave contributes half as much). Lower gain produces smoother, rounder terrain; higher gain produces rougher, spikier results.

Inigo Quilez's analysis [6] confirms that the standard gain of 0.5 (corresponding to Hurst exponent H=1) matches the spectral profile of real mountain profiles — a 9 dB/octave decay — making the parameters physically meaningful, not just aesthetic.

The first octave lays down the large-scale landform. The second adds hills. The third adds ridges. Each subsequent octave adds smaller and smaller wrinkles. A reader who stops at the first octave still sees a valid world; each additional octave is refinement.

---

## shaping terrain from a density field

With a noise function and fBm in hand, the simplest density function is:

```python
def terrain_density(x, y, z):
    surface_height = fbm(x, z)          # 2D fBm gives the base height
    return surface_height - y           # positive below, negative above
```

This gives a heightmap encoded as a density field — same limitation, different form. The density at position `(x, y, z)` is positive when `y` is below the surface and negative when above, so the iso-value of 0 gives the surface.

To get caves and overhangs, add 3D noise:

```python
def terrain_density_3d(x, y, z):
    base = fbm(x, z) - y                # base heightmap shape
    cave_carve = fbm(x * 2, y * 2, z * 2)   # 3D noise, finer scale
    return base - max(0, cave_carve - 0.3)  # subtract where cave noise is high
```

Wherever the 3D cave noise exceeds its threshold, density is subtracted — punching holes through the base terrain. The threshold and scale of the cave noise control how common and how large the caves are.

---

## carving structure — caves and tunnels

### 3D noise subtraction

The straightforward approach above — subtracting 3D noise from the base density — produces organic, blob-shaped voids that look like natural cave chambers. Adjusting the cave noise's lacunarity and frequency shapes whether caves are wide chambers or narrow passages.

### worm tunnels

A more controlled technique generates tunnels by tracing a path through the world. A "worm" walks from a seed point, steering its direction slightly at each step using noise (so the path curves organically), and carves out a cylindrical void along its path. Because the steering is noise-driven and the seed comes from the world coordinate, the tunnel is reproducible. The resulting caves are elongated corridors rather than blobs — useful for mineshaft or underground river aesthetics.

The two approaches compose: use worm tunnels for the main passages and 3D noise subtraction for side chambers and irregular alcoves.

---

## making worlds richer — domain warping and biome blending

### domain warping

Plain fBm produces recognisable fractal terrain, but it can look too regular — the same character everywhere. Domain warping breaks that uniformity by displacing the input coordinates before evaluating the noise.

Instead of sampling `fbm(p)`, compute a displacement vector at `p` using another fBm call, then sample `fbm(p + displacement)`. Inigo Quilez's article [7] describes the progression: a single warp produces swirling, river-valley-like features; nesting a second warp (`fbm(p + fbm(p + fbm(p)))`) adds more organic complexity at little extra cost.

The displacement breaks the lattice regularity of the base noise, producing results that look like erosion has been at work rather than a mathematical pattern.

### biome blending

Most interesting worlds vary by region — grasslands grade into deserts, temperate forests give way to snow. A <em>biome map</em> assigns each region a biome, typically using a separate low-frequency noise field (or Voronoi partitioning) to determine which biome a position belongs to.

Naively switching between biome density functions at hard boundaries produces visible seams. The solution is to evaluate multiple biome functions and blend between them — weighting each biome's contribution by a smooth falloff from the biome boundary. The blend width controls how gradual the transition looks.

---

## determinism and chunk streaming

The whole system is only viable for large worlds because every position regenerates identically. Given a <em>seed</em> — a large integer that initialises the pseudo-random number generator — the function `f(x, y, z, seed)` returns the same density every time, for any caller, anywhere in the world.

This matters enormously for [chunk management and streaming](../engines/chunk-management-and-streaming.md): the engine generates only the chunks near the player, discards them when the player moves away, and regenerates them on demand if the player returns — because the function is the ground truth, not the generated data. Two clients with the same seed build the same world independently, which is why Minecraft seeds are shareable.

The density function must be stateless and side-effect-free — a pure function. Any randomness must come from the seeded pseudo-random noise, not from global mutable state.

---

## tradeoffs and connections

Procedural terrain is cheap in storage and boundless in extent, but the world is only as varied as the density function allows. Hand-authoring specific shapes — a particular mountain, a ruin, a landmark — requires either placing authored SDFs on top of the procedural base (see [SDF and CSG modeling](./sdf-and-csg-modeling.md)) or hybrid approaches that blend procedural and authored content.

The output of the density function is a field of voxel classifications. To render it as smooth geometry rather than cubic blocks, it feeds a mesher — most commonly marching cubes ([marching cubes](../meshing/marching-cubes.md)), which extracts a smooth isosurface at the iso-value threshold. The density values also interact naturally with the data models explored in [voxel data models](../foundations/voxel-data-models.md).

Procedural terrain can extend beyond passive geometry: a cellular automaton seeded with the initial terrain state can simulate erosion, water flow, or settlement spread over time, building on the same coordinate-addressed grid — see [cellular automata](../simulation/cellular-automata.md).

The path from raw noise to complete world is: choose the terrain model (heightmap or density field) → define the density function using noise and fBm → add cave carving and domain warping → assign biomes → wrap in a deterministic seed → generate chunk by chunk on demand.

Every step is optional. A simple world works with only the first two. Richness comes from layering more structure onto the same foundation.

---

## references

[1] Perlin, K. (1985). "An image synthesizer." *Proceedings of SIGGRAPH 1985* (12th Annual Conference on Computer Graphics and Interactive Techniques). ACM. DOI: [10.1145/325334.325247](https://doi.org/10.1145/325334.325247) · [source](https://dl.acm.org/doi/10.1145/325334.325247)

[2] Perlin, K. (2002). "Improving noise." *Proceedings of ACM SIGGRAPH 2002*. ACM. DOI: [10.1145/566654.566636](https://doi.org/10.1145/566654.566636) · [source](https://dl.acm.org/doi/10.1145/566654.566636)

[3] Gustavson, S. (2005). "Simplex noise demystified." Technical report, Linköping University. [local PDF](../papers/gustavson-2005-simplex-noise-demystified.pdf) · [source](https://cgvr.cs.uni-bremen.de/teaching/cg_literatur/simplexnoise.pdf)

[4] McEwan, I., Sheets, D., Gustavson, S., and Richardson, M. (2012). "Efficient computational noise in GLSL." arXiv:1204.1461. [source](https://arxiv.org/abs/1204.1461)

[5] Fournier, A., Fussell, D., and Carpenter, L. (1982). "Computer rendering of stochastic models." *Communications of the ACM*, 25(6), 371–384. DOI: [10.1145/358523.358553](https://doi.org/10.1145/358523.358553) · [source](https://dl.acm.org/doi/10.1145/358523.358553)

[6] Quilez, I. (2023). "Fractal Brownian Motion." iquilezles.org. [source](https://iquilezles.org/articles/fbm/)

[7] Quilez, I. (2023). "Domain warping." iquilezles.org. [source](https://iquilezles.org/articles/warp/)
