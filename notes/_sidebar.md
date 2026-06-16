<!-- _sidebar.md — the two-scale onion. DOMAINS in dependency order, GLOBAL
     FOUNDATION first; nested **bold** = onion PHASE; links = pages. Absolute paths. -->

- **VOXEL FOUNDATIONS** <small>the shared mental model</small>
  - **foundation**
    - [what is a voxel](/foundations/what-is-a-voxel.md)
    - [voxels vs other representations](/foundations/voxels-vs-other-representations.md)
  - **building blocks**
    - [the voxel grid](/foundations/the-voxel-grid.md)
    - [what a voxel stores](/foundations/voxel-data-models.md)
  - **synthesis**
    - [the voxel pipeline](/foundations/the-voxel-pipeline.md)

- **STORING VOXELS** <small>data structures for the grid</small>
  - **foundation**
    - [the storage problem](/storing/the-storage-problem.md)
  - **building blocks**
    - [dense grids and chunks](/storing/dense-grids-and-chunks.md)
    - [octrees and sparse voxel octrees](/storing/octrees-and-svo.md)
    - [sparse voxel DAGs](/storing/sparse-voxel-dags.md)
    - [hash grids and brick maps](/storing/hash-grids-and-bricks.md)
    - [OpenVDB and NanoVDB](/storing/openvdb-and-nanovdb.md)
  - **cross-cutting**
    - [choosing a voxel store](/storing/choosing-a-voxel-store.md)

- **GENERATING & VOXELIZING** <small>where the data comes from</small>
  - **foundation**
    - [where voxels come from](/generating/where-voxels-come-from.md)
  - **building blocks**
    - [procedural terrain](/generating/procedural-terrain.md)
    - [SDF and CSG modeling](/generating/sdf-and-csg-modeling.md)
    - [mesh voxelization](/generating/mesh-voxelization.md)
    - [scanned and volume data](/generating/scanned-and-volume-data.md)

- **SURFACE EXTRACTION** <small>turning a field into a surface</small>
  - **foundation**
    - [why mesh voxels](/meshing/why-mesh-voxels.md)
  - **building blocks**
    - [blocky and greedy meshing](/meshing/blocky-and-greedy-meshing.md)
    - [marching cubes](/meshing/marching-cubes.md)
    - [surface nets and dual contouring](/meshing/surface-nets-and-dual-contouring.md)
  - **cross-cutting**
    - [LOD seams and transvoxel](/meshing/lod-seams-and-transvoxel.md)
    - [choosing a meshing algorithm](/meshing/choosing-a-meshing-algorithm.md)

- **RENDERING VOXELS** <small>getting voxels onto the screen</small>
  - **foundation**
    - [ways to render voxels](/rendering/ways-to-render-voxels.md)
  - **building blocks**
    - [grid ray traversal](/rendering/grid-ray-traversal.md)
    - [sparse voxel octree raytracing](/rendering/sparse-voxel-octree-raytracing.md)
    - [volume ray casting](/rendering/volume-ray-casting.md)
    - [splatting and point rendering](/rendering/splatting-and-point-rendering.md)
  - **cross-cutting**
    - [choosing a render path](/rendering/choosing-a-render-path.md)

- **VOXEL ENGINES IN PRACTICE** <small>assembling a running system</small>
  - **foundation**
    - [anatomy of a voxel engine](/engines/anatomy-of-a-voxel-engine.md)
  - **building blocks**
    - [chunk management and streaming](/engines/chunk-management-and-streaming.md)
    - [threading and the meshing pipeline](/engines/threading-and-meshing-pipeline.md)
    - [runtime editing and CSG](/engines/runtime-editing-and-csg.md)
  - **cross-cutting**
    - [LOD in engines](/engines/lod-in-engines.md)
  - **synthesis**
    - [case studies](/engines/case-studies.md)

- **OPTIMIZATION & PERFORMANCE** <small>making it fast and small</small>
  - **foundation**
    - [the performance budget](/optimization/the-performance-budget.md)
  - **building blocks**
    - [memory layout and Morton order](/optimization/memory-layout-and-morton.md)
    - [compression techniques](/optimization/compression-techniques.md)
    - [LOD and culling](/optimization/lod-and-culling.md)
    - [GPU voxel techniques](/optimization/gpu-voxel-techniques.md)
  - **cross-cutting**
    - [baking ambient occlusion and light](/optimization/baking-ambient-occlusion-and-light.md)

- **SIMULATION ON VOXEL GRIDS** <small>the grid as a physics substrate</small>
  - **foundation**
    - [voxels as a simulation grid](/simulation/voxels-as-a-simulation-grid.md)
  - **building blocks**
    - [cellular automata](/simulation/cellular-automata.md)
    - [light propagation](/simulation/light-propagation.md)
    - [voxel physics and destruction](/simulation/voxel-physics-and-destruction.md)
    - [fluid and smoke](/simulation/fluid-and-smoke.md)

- **ADVANCED APPLICATIONS** <small>voxels beyond games</small>
  - **foundation**
    - [voxels beyond games](/applications/voxels-beyond-games.md)
  - **building blocks**
    - [voxel global illumination](/applications/voxel-global-illumination.md)
    - [scientific and medical volume rendering](/applications/scientific-and-medical-volume-rendering.md)
    - [VDB in VFX](/applications/vdb-in-vfx.md)
    - [machine learning on voxels](/applications/machine-learning-on-voxels.md)
    - [voxels in fabrication](/applications/voxels-in-fabrication.md)

- **&nbsp;**
  - [search](/search.md)
