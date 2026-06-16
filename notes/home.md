<link rel="stylesheet" href="./css/globals.css">

# voxels & voxel algorithms

study notes on representing the world as a 3D grid of cells — and every algorithm
that stores, builds, meshes, renders, and simulates that grid. built to give you a
<em>mental model</em> first and the fine print last, so you can stop at any depth and
still understand the shape of things.

## how these notes are structured

two onions. the **domains** down the sidebar go in dependency order — each rests on
the ones above it — and within each domain the pages peel from a coarse mental model
inward to specifics.

the spine follows the life of voxel data:

1. **voxel foundations** — what a voxel is, and why you'd use a grid at all.
2. **storing voxels** — fitting a 3D grid in memory without it exploding.
3. **generating & voxelizing** — where the data comes from.
4. **surface extraction** — turning the grid into a surface you can draw.
5. **rendering voxels** — getting it onto the screen (mesh, ray-march, or volume).
6. **voxel engines in practice** — assembling the pieces into a running system.
7. **optimization & performance** — making it fast and small.
8. **simulation on voxel grids** — using the grid as a physics substrate.
9. **advanced applications** — global illumination, medical viz, VFX, ML, fabrication.

## where to start

- new to voxels? read **voxel foundations** top to bottom, then follow the sidebar.
- building an engine? foundations → storing → surface extraction → rendering → engines.
- here for one algorithm? jump straight to its page, or use [search](./search.md).

## on sources

content pages cite open-access papers inline and keep local PDFs of the referenced
work under `papers/` so the primary sources travel with the notes.
