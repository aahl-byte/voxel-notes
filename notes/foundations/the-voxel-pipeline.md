<link rel="stylesheet" href="./css/globals.css">

# the voxel pipeline

Every voxel system — a destructible game world, a CT scan viewer, a VFX volume simulation — is the same handful of stages wired together. The stages vary in how they're implemented and which algorithms they call, but the shape of the problem is always the same: hold the data, fill it with something, turn it into something drawable, then draw it. These notes are organized around that lifecycle.

---

## the lifecycle at a glance

A voxel system does four things, in order, and often in a loop:

1. **store** — hold the grid in memory without it collapsing under its own size.
2. **generate** — fill the grid: write values into cells that describe what occupies space.
3. **extract** — decide how to make the field drawable: either pull a surface mesh out of it, or leave it as a volume and ray-march through it directly.
4. **render** — get pixels on screen: rasterize the mesh, ray-march the grid, or composite it as a volume.

That's the whole spine. Every page in these notes lives somewhere on it.

---

## the four stages, described

### store — keeping the grid alive

The naive grid is a uniform 3-D array: every cell takes the same memory slot whether it holds rock, air, or water. That works at small scale. At game-world or medical-dataset scale it explodes — a 1024³ grid of 4-byte floats is 4 GB before you've done anything with it.

So the first job of any voxel system is <em>choosing a representation that trades memory for what you actually need</em>. Sparse structures skip empty space entirely. Hierarchical structures let you store coarse detail far from the camera and fine detail up close. Compressed formats pack runs of identical values. The choice made here — flat array, octree, hash map, VDB — propagates downstream and constrains every other stage.

This is the subject of [the storage problem](../storing/the-storage-problem.md), and the data model options are laid out in [voxel data models](./voxel-data-models.md).

---

### generate — filling the grid

Before a voxel system can do anything visible, values have to get into cells. There are four broad ways this happens:

- **procedural** — a function maps a 3-D coordinate to a value. Noise functions, signed-distance fields, and layered rules produce terrain, caves, and organic shapes without any artist-authored data. Fast, infinitely scalable, but only as rich as the function.
- **modeled** — an artist places voxels directly in a voxel editor, or a mesh authored in conventional 3-D tools is <em>voxelized</em> — converted into voxel form by testing which cells the mesh's triangles occupy. Goes from geometry to field.
- **scanned** — real-world capture devices (CT scanners, lidar rigs, depth cameras) produce density or distance measurements that map naturally onto voxel grids. Goes from the physical world to field.
- **simulated** — a running simulation writes new values into the grid each frame. Fluid, fire, and destruction dynamics live here.

These paths are not mutually exclusive — a terrain system might start procedural, let an artist sculpt over it, then simulate erosion on top.

The generating domain starts at [where voxels come from](../generating/where-voxels-come-from.md).

---

### extract — making the field drawable

A voxel field on its own is not a renderable object. It's a 3-D array of numbers. To get it on screen you have to decide: do you want a surface, or do you want to render the field directly?

**If you want a surface (meshing path):**

Pull a polygon mesh out of the field by finding where the values cross a threshold — the boundary between "inside" and "outside." The output is ordinary triangles the GPU already knows how to rasterize. Algorithms for this include:

- *marching cubes* — steps through every cell, looks up a table based on which corners are inside/outside, places triangles.
- *dual contouring* — uses gradient information at crossings to position vertices, preserving sharp features marching cubes rounds off.
- *transvoxel* — an extension of marching cubes that stitches crack-free seams between chunks at different levels of detail.

The meshing approaches are compared in [why mesh voxels](../meshing/why-mesh-voxels.md).

**If you want to skip meshing (direct render path):**

Send rays through the grid from the camera and accumulate samples as each ray travels through occupied cells. No mesh is ever produced. This is <em>ray marching</em> — cheap to set up, expensive per pixel, and the only way to render soft volumetric effects (fog, fire, clouds) without approximation. Medical volume renderers almost always take this path.

---

### render — getting pixels on screen

After extraction the system has either a mesh or is ray-marching directly. Either way, rendering turns that into a 2-D image:

- **rasterize the mesh** — hand standard triangle geometry to the GPU's rasterizer. Fast, familiar, plays well with the rest of a game's render pipeline.
- **ray-march the grid** — cast rays per pixel through the voxel volume; accumulate color and opacity along each ray. Handles transparency and soft volumes naturally, but the cost scales with ray depth and grid density.
- **volume composite** — the classical medical-imaging approach: sort and blend voxel slabs front-to-back or back-to-front, applying transfer functions that map density values to color and opacity.

The rendering domain covers all three paths in [ways to render voxels](../rendering/ways-to-render-voxels.md).

---

## the cross-cutting wrappers

The four-stage core is the spine, but a real system adds three concerns that wrap around it:

### assemble it into a running system

An engine coordinates the stages: streaming chunks in and out as the camera moves, scheduling generation and extraction jobs on background threads, managing the scene graph and input. That architecture is the subject of [anatomy of a voxel engine](../engines/anatomy-of-a-voxel-engine.md).

### make it fast and small

The core loop is expensive. Voxel counts are enormous; meshing is CPU-heavy; ray marching is GPU-heavy. <em>Optimization</em> cuts across every stage — picking better data structures (storage), culling empty regions before meshing (extraction), reducing overdraw (rendering). The tradeoffs are mapped in [the performance budget](../optimization/the-performance-budget.md).

### let the grid simulate

When the voxel field is writable at runtime, the grid becomes a simulation substrate: fluids flow through cells, fire propagates outward, structural integrity collapses inward. The simulation loop writes new values into the same grid the render loop reads. That feedback is explored in [voxels as a simulation grid](../simulation/voxels-as-a-simulation-grid.md).

---

## it's a cycle, not a straight line

The diagram looks like a pipeline, but in practice it loops:

```
store ──► generate ──► extract ──► render
  ▲                                   │
  └─────── edit / simulate ───────────┘
```

When a player destroys a wall, the generation stage writes new values; the extraction stage re-meshes the affected chunk; the render stage draws the updated mesh. The data model chosen in the store stage determines how fast that loop can run. A flat array re-meshes trivially but can't skip empty space. A sparse octree skips empty space but requires tree surgery on every edit.

This is why [what is a voxel](./what-is-a-voxel.md) and [voxel data models](./voxel-data-models.md) matter before anything else — the foundation constrains the whole cycle.

The same loop appears outside games. CT and MRI viewers fill the grid from scanner data, then ray-march it directly — no meshing step. VFX pipelines voxelize a simulation, extract a surface at each frame, and render with path tracing. Robotics systems fill the grid from lidar, then query it for occupancy rather than rendering it at all. The applications domain surveys this breadth in [voxels beyond games](../applications/voxels-beyond-games.md).

---

## how to read these notes

The site is organized to follow the lifecycle, domain by domain. There's no single right order — pick the path that matches what you're trying to do.

### new to voxels

Start at the foundations and read forward in sidebar order:
1. [what is a voxel](./what-is-a-voxel.md) — the primitive and why it matters.
2. [voxel data models](./voxel-data-models.md) — the representations that make scale possible.
3. This page — the whole lifecycle as a map.
4. Then follow whichever domain interests you first.

### building an engine

You'll want the full spine before diving deep:
1. Foundations → Storing → Generating → Meshing → Rendering → Engines, in that order.
2. Then Optimization, since budget decisions ripple back through every stage.
3. Simulation if your engine needs it; Applications for context.

### studying a single algorithm

Jump directly to the domain that owns it. Each domain page is written to be coherent on its own, and cross-links will surface the prerequisite ideas as you encounter them.

---

The rest of these notes flesh out each stage. This page is the map — when you get lost, come back here.
