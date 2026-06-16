<link rel="stylesheet" href="./css/globals.css">

# light propagation

Place a torch in a dark cave in Minecraft and something satisfying happens: a warm glow spreads outward, fading as it reaches the cave walls, casting shadows in every nook. Break a block in the ceiling and sunlight pours through, filling the space below. Place another torch nearby and the two pools of light blend together.

This is the goal of flood-fill voxel lighting. It has to update in real time as the player edits the world, it has to handle both torches (point sources) and sunlight (the sky above), and it has to do it cheaply enough to run every frame. Understanding how it works means understanding one central question: given that light values live on a grid of voxels, how do you spread a value outward from a source and then correctly undo that spread when something changes?

The voxel grid and its neighbor-lookup properties that make this tractable are covered in [voxels as a simulation grid](./voxels-as-a-simulation-grid.md).

---

## the coarse model

Light in a voxel world is a number, not a ray. Each voxel stores a light level — in Minecraft's case, an integer from 0 (pitch black) to 15 (maximum brightness). When a torch is placed, it seeds its voxel with a value of 14 (one below the maximum, which is reserved for sunlight). The engine then visits every air voxel neighboring the torch, writes 13 into it, then visits every air voxel neighboring those voxels and writes 12, and so on, stopping when the value would drop to zero or it hits a solid block. This is a <em>flood fill</em> — the same breadth-first expansion used in paint-bucket tools, just running in 3D across six face-neighbors per cell.

A voxel gets 1 subtracted per step. Solid blocks stop the flood. The result is a smooth gradient of integers that falls off with taxicab distance from the source [1][2].

That is the whole mental model. Everything else is solving for edge cases: sunlight behaves slightly differently, colored light needs multiple channels, and removing a torch is much harder than placing one.

---

## the two light systems

### block light — point sources

Block light is the simpler case. Any light-emitting block — a torch, glowstone, a campfire — seeds its voxel with an emission value (1–15). The flood fill radiates outward, decrementing by 1 per step through air and transparent blocks, halting at solid voxels [1].

- Taxicab (Manhattan) distance rules: diagonal voxels cost the sum of axis distances, not the Euclidean distance.
- Multiple overlapping sources blend naturally: each voxel simply holds the maximum value any source has propagated into it.
- Colored torches use separate red, green, and blue channels — more on this below.

### skylight — sunlight from above

Skylight starts from a different premise. The engine tracks a <em>heightmap</em> for each X/Z column: the Y position of the highest solid block. Every voxel above the heightmap is in full sky — level 15. Here is where skylight behaves differently from a torch: when it spreads straight down through open air, it does not decrement. A column of open air from sky to floor is uniformly at level 15 all the way down [1][2].

Once skylight spreads sideways — into a cave, around a corner — it decrements by 1 per horizontal step, just like block light. So the first voxel of cave air adjacent to the open column is at 14, the next is 13, and so on.

This vertical-pass-through rule is what gives Minecraft its characteristic bright shafts of light: breaking a ceiling block fills the entire vertical space beneath it with full daylight, which then fades laterally into the cave.

---

## the propagation algorithm

The flood fill runs as a standard BFS queue. Placing a torch triggers it; the same loop handles both block light and skylight initialization.

**Conceptually, each queue entry holds a voxel position and the light level arriving at that voxel.** The loop:

1. Pop the front entry from the queue.
2. For each of the six face-neighbors:
   - Skip if solid.
   - Skip if the neighbor's current light level is already ≥ (current level − 1). It has already been reached by a brighter source.
   - Otherwise: write (current level − 1) into the neighbor, and push that neighbor onto the queue.
3. Repeat until the queue is empty.

For skylight propagating downward at full strength, step 2 is modified: the − 1 decrement is skipped, so the neighbor receives the same level as the current voxel [2].

```python
# BFS flood fill — block light example
# Each entry: (voxel_index, light_level)
queue = deque()
queue.append((torch_index, 14))
grid[torch_index].block_light = 14

while queue:
    idx, level = queue.popleft()
    for neighbor_idx in face_neighbors(idx):
        if is_solid(neighbor_idx):
            continue
        new_level = level - 1
        if new_level <= grid[neighbor_idx].block_light:
            continue          # already lit by a brighter source
        grid[neighbor_idx].block_light = new_level
        if new_level > 1:
            queue.append((neighbor_idx, new_level))
```

BFS is the right traversal here, not DFS. BFS visits voxels in order of increasing distance from the source, which guarantees that when a voxel is first dequeued, it holds the maximum value any source could ever write into it. DFS would visit voxels in arbitrary order and require multiple passes to converge [3].

---

## the hard part — removal

Adding a light source is easy. Removing one is not.

When a torch is broken, the engine cannot simply re-run the flood fill from scratch for the entire world — that would be too slow. It needs to identify and erase only the light values that came from the removed torch, then re-propagate from any remaining sources that can reach the now-dark region.

The standard solution is a two-pass BFS [2][4]:

### pass 1 — the removal sweep

1. Push the now-empty torch voxel onto a **removal queue**, carrying the light level it used to hold.
2. Pop the front entry. For each face-neighbor:
   - If the neighbor's light level is nonzero and strictly less than the current entry's level, that neighbor was lit by this source (a brighter source would have overwritten it). Zero it out and push it onto the removal queue.
   - If the neighbor's light level is ≥ the current entry's level, it is lit by a different source. Push it onto the **propagation queue** (pass 2 will re-fill from there).
3. Continue until the removal queue is empty.

### pass 2 — re-propagation

Run the standard BFS flood fill starting from everything in the propagation queue. These are the surviving sources and source-adjacent voxels that can re-illuminate the now-dark region.

The removal sweep is the more subtle of the two. It exploits the invariant that flood fill always writes the maximum reachable level: if a neighbor's value is less than what the current source provided, it could only have gotten that value from this source, so it belongs in the removal pass. If the neighbor's value is higher, something else lit it, so re-propagation should start from there [2][4].

```python
# Two-pass removal
removal_queue = deque()
propagation_queue = deque()

old_level = grid[torch_index].block_light
grid[torch_index].block_light = 0
removal_queue.append((torch_index, old_level))

# Pass 1: sweep stale light
while removal_queue:
    idx, level = removal_queue.popleft()
    for neighbor_idx in face_neighbors(idx):
        neighbor_level = grid[neighbor_idx].block_light
        if neighbor_level == 0:
            continue
        if neighbor_level < level:
            # This voxel was lit by the removed source
            grid[neighbor_idx].block_light = 0
            removal_queue.append((neighbor_idx, neighbor_level))
        else:
            # Different source — seed re-propagation
            propagation_queue.append((neighbor_idx, neighbor_level))

# Pass 2: re-fill from surviving sources
bfs_flood_fill(propagation_queue)
```

The same two-pass structure applies to skylight: blocking a window in the ceiling triggers a removal sweep downward and sideways through the shadow, followed by re-propagation from the edges that still reach daylight [2].

---

## colored light — RGB channels

White torches are a single integer. Colored torches — a red lantern, a blue portal — require multiple values per voxel.

The approach: store separate light levels for each channel. In a 2-byte layout, one byte can hold R/G/B as three 4-bit fields (0–15 each), and the remaining 4 bits hold skylight — `SSSS RRRR GGGG BBBB` [2]. The flood fill runs independently on each channel, which means colored sources blend automatically: an orange torch `(R=15, G=8, B=0)` eventually produces a dim red `(R=2, G=1, B=0)` at distance, since the channels attenuate at different rates.

One known limitation of per-channel independent propagation: a source like `(R=15, G=8, B=0)` will, at sufficient distance, become purely red because the green channel hits zero sooner. This color-shift-with-distance can be a feature (warm near the torch, cooler at range) or a visual artifact, depending on the palette [2].

Colored glass filters are handled during propagation: each step multiplies the outgoing level by the glass block's per-channel transmittance value. Red glass at transmittance `(1.0, 0.0, 0.0)` reduces green and blue to zero immediately, producing a colored shaft of light.

---

## smooth lighting — handing values to the mesher

The voxel grid holds per-voxel integer light levels. The rendered image needs per-pixel lighting. The bridge is the mesher.

When the [mesher](../meshing/blocky-and-greedy-meshing.md) generates a quad for a visible face, it samples the light levels from the four voxels surrounding each vertex (the face voxel itself and its face-diagonal neighbors). It averages these into a single floating-point brightness value per vertex and embeds it in the vertex data. The GPU then interpolates linearly across the quad, producing a smooth lighting gradient without any additional cost at render time.

This is Minecraft's "smooth lighting" mode. It gives the characteristic gradient across lit faces that makes the BFS approximation look much more convincing than raw per-face flat shading [5].

The vertex-level sampling interacts directly with ambient occlusion. AO values are computed from the same neighborhood — the three voxels that would block the vertex's corner — and combined with the light level in the vertex data. The mesher handles both at once. This is covered in detail in [baking ambient occlusion and light](../optimization/baking-ambient-occlusion-and-light.md).

The key principle: the light grid is computed once by the BFS system, and the mesher consumes those values at mesh-build time. Updating a chunk's mesh re-samples the grid; the grid itself is not re-sampled at render time.

---

## performance — dirty regions and incremental updates

Running a full-world BFS on every block change would be impractical. Voxel engines constrain the flood fill to <em>dirty regions</em> — the set of chunks whose lighting could have changed.

The rules are straightforward [6][7]:

- A block change dirties its own chunk.
- Because light can propagate across chunk borders, the eight neighboring chunks (in a 2D chunk grid) or the surrounding 26 (in 3D) must also be marked dirty.
- Only dirty chunks are re-lit; clean chunks are skipped entirely.

The removal sweep naturally limits itself: the BFS terminates as soon as all affected voxels have been zeroed, which rarely spans more than a few chunks for a typical torch. Skylight columns can be longer, but the heightmap caps them: once the sweep reaches a height above the heightmap, it stops — those voxels are never shaded by block sources.

The link to runtime editing is direct. The [runtime editing and CSG](../engines/runtime-editing-and-csg.md) page covers how engines track which chunks need geometry rebuilds; voxel lighting plugs into the same dirty-flag system. A block placement triggers: (1) update the voxel grid, (2) mark affected chunks dirty in both the lighting system and the meshing pipeline, (3) re-light dirty chunks with BFS, (4) rebuild dirty chunk meshes with the updated light values. Steps 3 and 4 can be pipelined or run on background threads.

Some engines further batch pending light updates — deferring them until the affected region is actually queried for rendering — to avoid redundant re-lighting when multiple edits land in the same region within a single frame [7].

---

## contrast — BFS propagation vs. true global illumination

BFS flood fill is cheap and deterministic, but it is an approximation. It captures the main things that make spaces feel lit: brightness falls with distance, solid blocks cast hard shadows, and large open areas feel brighter than tunnels. What it does not capture:

| property | BFS flood fill | voxel cone tracing / GI |
|---|---|---|
| light bouncing off surfaces | no | yes |
| soft indirect illumination | no | yes |
| color bleeding between surfaces | no | yes |
| specular highlights | no | yes |
| performance (per frame) | very cheap | expensive |
| runtime edits | fast incremental | re-voxelize + re-trace |
| implementation complexity | low | high |

<em>Voxel cone tracing</em> — the technique introduced by Crassin et al. [8] — represents the expensive end of this spectrum. It voxelizes the scene into a sparse octree, then traces wide cones through that octree to gather incoming radiance from multiple bounces. The result is physically plausible indirect illumination: light that reflects off a red wall and tints the ceiling, soft penumbrae, and glossy reflections. This is covered in [voxel global illumination](../applications/voxel-global-illumination.md).

For open-world games where players edit the terrain in real time, BFS flood fill remains the practical default. The cost difference is not marginal: cone tracing requires a full scene re-voxelization and an octree mipmap rebuild on every significant edit, while BFS needs only a localized two-pass queue sweep. The use case governs the choice:

- **use BFS flood fill when:** the world is editable, you need real-time block updates, and approximate lighting is acceptable.
- **use voxel cone tracing / GI when:** the scene is mostly static, you need accurate light bouncing, and you have GPU budget to spend.

---

## the specifics

### storage layout

Per-voxel light data is typically packed into 1–4 bytes alongside the block type:

- **1 byte (Minecraft-style):** 4 bits skylight + 4 bits block light. Monochromatic only.
- **2 bytes (Seed of Andromeda-style):** 4 bits skylight + 4 bits R + 4 bits G + 4 bits B. Colored light at modest memory cost [2].
- **4 bytes (extended):** additional directional skylight components for multi-angle sun simulation [3].

Light data is separate from the block type. Voxels in the lighting array are indexed identically to the block array — same linearized `x + y*W + z*W*H` layout — so neighbor lookups are constant-time array offsets.

### queue entry representation

Each BFS queue entry stores a linearized voxel index (a single integer, not an x/y/z triple) and the light level being propagated. Linearized indices are faster: one integer comparison instead of three, and better cache behavior when the queue is processed in order [2].

### max-level sunlight propagation

When skylight at level 15 propagates downward, the decrement is suppressed: the neighbor is written as 15, not 14. This requires a special check in the propagation loop. The removal pass mirrors this: when zeroing a downward-propagating column at level 15, it always removes the voxel below, even if that voxel holds a value less than 15 from another source — because the downward-propagation rule would have overwritten any lower value anyway [2].

### chunk boundary handling

Light can propagate across chunk boundaries. The standard approach: when re-lighting a chunk, seed the BFS queue not only from internal sources but also from border voxels of adjacent chunks that have non-zero light levels. The PocketMine lighting spec formalizes this as three passes: discover internal sources, propagate within the chunk, then propagate across borders in groups of adjacent chunks [6].

---

## references

[1] Minecraft Wiki. "Light." Retrieved June 2026. [source](https://minecraft.wiki/w/Light)

[2] Arnold, B. (Seeds of Andromeda). "Fast Flood Fill Lighting in a Blocky Voxel Game, Parts 1 & 2." seedofandromeda.com, 2014. [Part 1](https://www.seedofandromeda.com/blogs/29-fast-flood-fill-lighting-in-a-blocky-voxel-game-pt-1) · [Part 2](http://www.seedofandromeda.com/blogs/30-fast-flood-fill-lighting-in-a-blocky-voxel-game-pt-2) (Mirrored: [notverymoe.github.io](https://notverymoe.github.io/md-gamedev-gems/voxel/lighting/soa/index.html))

[3] 0fps. "Voxel Lighting." 0fps.net, February 2018. [source](https://0fps.net/2018/02/21/voxel-lighting/)

[4] dktapps. "Lighting Algorithm Specification." GitHub, 2019. [source](https://github.com/dktapps/lighting-algorithm-spec)

[5] 0fps. "Ambient Occlusion for Minecraft-Like Worlds." 0fps.net, July 2013. [source](https://0fps.net/2013/07/03/ambient-occlusion-for-minecraft-like-worlds/)

[6] dktapps. "Lighting Algorithm Specification — PocketMine three-pass approach." GitHub. [source](https://github.com/dktapps/lighting-algorithm-spec)

[7] greyminecraftcoder. "Minecraft Modding: Lighting." Blogger, 2013. [source](http://greyminecraftcoder.blogspot.com/2013/08/lighting.html)

[8] Crassin, C., Neyret, F., Sainz, M., Green, S., and Eisemann, E. (2011). "Interactive Indirect Illumination Using Voxel Cone Tracing." *Computer Graphics Forum*, 30(7), 1921–1930. DOI: 10.1111/j.1467-8659.2011.02063.x. [local PDF](../papers/crassin-2011-voxel-cone-tracing.pdf) · [source](https://research.nvidia.com/publication/2011-09_interactive-indirect-illumination-using-voxel-cone-tracing)
