<link rel="stylesheet" href="./css/globals.css">

# scanned and volume data

A surgeon plans an operation by navigating through a patient's organs before making the first incision. A robot builds a map of a warehouse in real time, distinguishing free corridors from shelving. A handheld depth camera reconstructs a room from a minute of walking around it. In each case, the goal is the same: fill a voxel grid with data about the real world, so a system can reason about what is where.

This page covers four routes from measured reality to a populated grid — medical imaging, point clouds, depth fusion, and robotics occupancy — and notes where each kind of filled grid goes next.

---

## the coarse model

Every method here follows the same three-step logic:

1. A sensor samples the physical world at discrete locations — X-ray attenuation along each CT beam, range distances from a depth camera, reflected pulses from a lidar.
2. Those samples are mapped into a 3D grid. The mapping may be direct (CT slices already form a grid) or require binning, integration, or probabilistic update.
3. Each cell ends up holding a value that means something specific to the application — a calibrated tissue density, a signed distance to the nearest surface, a probability of occupancy.

The output in all four cases is a voxel grid. What varies is the type of value per cell, and what that value is useful for.

If you are not already comfortable with how a voxel grid maps between world coordinates and cell indices, read [the voxel grid](../foundations/the-voxel-grid.md) first — the affine transform and anisotropic spacing concepts there are the mechanical foundation for everything on this page.

For a broader map of where scanning and filling fits in the pipeline, see [where voxels come from](./where-voxels-come-from.md).

---

## medical imaging — the native voxel grid

### the outcome

A radiologist needs to identify a tumor's boundaries. A neurosurgeon needs to know, pre-operatively, which vessels run through the tissue they plan to resect. In both cases, the instrument they reach for produces a volume: a 3D grid of tissue-density values ready to navigate, segment, and measure.

### how a CT scan becomes a voxel grid

A CT scanner rotates an X-ray source around the patient while detectors on the opposite side measure how much of the beam each column of tissue absorbed. Reconstruction algorithms (filtered back-projection or iterative methods) turn those projections into a 2D slice image. The scanner acquires hundreds of these slices along the body axis, each a short distance from the last. Stack those slices and you have a 3D array — a voxel grid — where each cell holds the X-ray attenuation at that location in the body.

There is no conversion step. The CT scanner natively produces a 3D grid of samples. The structure IS a voxel volume.

The value stored at each cell is a <em>Hounsfield unit (HU)</em>: a calibrated measure of how much the tissue at that location attenuates X-rays relative to water. The scale is fixed by two anchors:

- water = 0 HU (the physical calibration reference)
- air = −1000 HU

Tissue types sort onto the scale predictably:

| tissue | typical HU range |
|---|---|
| air | −1000 |
| lung (air-filled) | −900 to −500 |
| fat | −100 to −50 |
| soft tissue | +20 to +80 |
| blood / contrast | +40 to +90 |
| bone (cortical) | +400 to +1900 |
| metal implant | +1000 and above |

Because the scale is standardized, a windowing algorithm can extract exactly the tissue type it cares about by thresholding: set any cell below +200 HU to transparent and above +1000 HU to white, and you get a bone-only view. Swap the window and you visualize soft tissue instead. The calibration is what makes this reliable across scanners and patients.

MRI produces a grid the same way — slices stacked into a volume — but the values are signal intensities that depend on the imaging sequence rather than a single universal scale. The grid structure is identical; the physical meaning of the number per cell differs.

### anisotropic spacing — the important catch

CT grids are almost always anisotropic. The in-plane resolution (within each slice) is controlled by detector pitch and field of view; clinical scanners typically achieve 0.5–0.9 mm per pixel. The spacing *between* slices — set by how fast the table moved and how many slices the protocol requested — is often 1–5 mm, sometimes more.

A volume acquired at 0.7 × 0.7 mm in-plane but 3 mm between slices has voxels shaped like flat tiles, not cubes. Code that assumes equal spacing in all three axes will compute wrong distances, wrong surface normals, and wrong gradient directions from this data.

The spacing values travel with the data as DICOM metadata:
- `PixelSpacing` (tag 0028,0030) — row and column spacing within each slice, in mm
- `ImagePositionPatient` (tag 0020,0032) — the world position of the first pixel of each slice
- `ImageOrientationPatient` (tag 0020,0037) — direction cosines for the row and column axes

From these three fields you reconstruct the exact 4×4 affine matrix that maps any voxel index (i, j, k) to its position in millimetre-accurate patient space — the same transform described in [the voxel grid](../foundations/the-voxel-grid.md). The slice spacing is derived from the difference between consecutive `ImagePositionPatient` values along the normal axis.

Any algorithm that needs isotropic voxels (marching cubes for surface extraction, for instance) must resample the volume to equal spacing as a preprocessing step, or explicitly account for per-axis spacing in every distance calculation.

---

## point clouds → voxels

### the outcome

A lidar sensor on a self-driving vehicle returns tens of millions of unordered 3D points per second. A laser scanner surveying a building returns a billion points. Neither structure is directly usable for simulation or real-time queries — the points are irregular, variable-density, and carry no adjacency information. Converting to a voxel grid regularizes the data into a structure that downstream code can index in O(1) and iterate uniformly.

### how it works — voxel downsampling

The simplest conversion is called <em>voxel downsampling</em>: choose a cell size, then for each point compute its grid index using the world→grid formula, and bin the point into that cell.

```
cell = floor((point - grid_origin) / voxel_size)
```

Every cell accumulates the points that fall into it. Per cell, you then record:

- **occupancy** — the cell is non-empty (binary; the coarsest form)
- **centroid** — the average 3D position of all points in the cell
- **average color or normal** — computed from the points' attributes

The result is a regular grid where each occupied cell represents the local average of all raw samples. The grid is typically sparse — most cells are empty — which is why point-cloud workflows often store it as a hash grid or octree rather than a dense array. See [the storage problem](../storing/the-storage-problem.md) for why dense allocation is often impractical here.

### what is lost, and why that is often fine

Downsampling is lossy: fine detail smaller than one cell is gone, and the original point distribution within each cell is replaced by a single representative. What remains is a regular structure:

- neighbor queries are O(1) index offsets instead of spatial tree traversals
- simulation and ML algorithms that sweep the grid iterate uniformly, with no variable-density artifacts
- memory per occupied region is bounded and predictable

The tradeoff is cell size. Larger cells = faster queries, less detail. Smaller cells = more cells, more storage, finer fidelity. There is no free lunch; the choice is the same tradeoff that governs any discretization.

For comparison with surface-based voxelization from a mesh, see [mesh voxelization](./mesh-voxelization.md).

---

## depth fusion — the TSDF

### the outcome

A handheld depth camera (RGB-D sensor) captures a noisy, partial depth image thirty times per second. No single frame gives a complete 3D model — the frame is partial, and each measurement is corrupted by sensor noise. The goal is to fuse many frames into a single, clean, dense 3D reconstruction.

### describe the idea, then name it

Imagine a 3D grid overlaid on the scene. For each depth pixel in each frame, you can project a ray from the camera through that pixel into the grid and note: for cells along that ray that are *in front of* the measured surface, the surface is some positive distance away; for cells *behind* the measured surface, the distance is negative. If you do this for hundreds of frames and average the distance values per cell, the noise cancels and the true surface is where the average value is zero — a sign change from positive to negative.

This is a <em>truncated signed distance field (TSDF)</em>: a volume where each cell holds the signed distance to the nearest surface, positive in free space and negative behind it, and *truncated* so only the cells within a narrow band around the surface are updated.

The truncation is what makes it practical. If a cell is far from any surface in the current frame, skip it. Only update the narrow band (cells within ±δ of the measured surface). This keeps the computation cost proportional to the surface area, not the full volume.

Curless and Levoy formalized this approach in 1996 for fusing range images from a structured-light scanner [[1]](#references). The fusion rule per cell is a running weighted average:

```
D_new = (W_old × D_old + w × d) / (W_old + w)
W_new = W_old + w
```

where `d` is the signed distance from the current frame, `w` is a per-observation confidence weight (often based on the angle of incidence or inverse distance), and `D_old`/`W_old` are the accumulated value and weight. The surface reconstructed from the TSDF is the isosurface where D = 0, extracted by marching cubes or similar.

### kinectfusion — real-time TSDF fusion

Newcombe et al. at Microsoft Research brought this to real time in 2011 [[2]](#references). KinectFusion ran the full TSDF pipeline — frame integration, camera tracking, and surface prediction — on GPU hardware, using a consumer depth camera (the Xbox Kinect). The system updates the TSDF with each new 512×424 depth frame at 30 Hz, tracking the camera pose by aligning each incoming frame against the current surface prediction via iterative closest point (ICP). The result is a dense, clean reconstruction from a minute of walking around a room.

The key insight is that the TSDF handles sensor noise automatically: noise averages out across frames because each cell's value is the weighted mean of many independent measurements. A single frame's outliers cannot corrupt the global model; they are diluted by the accumulated weight.

#### TSDF vs simple occupancy for fusion

When fusing many noisy depth frames into a volume:

- **binary occupancy** per cell: each new frame votes a cell occupied or free. Noise causes cells near the surface to flicker, and there is no graceful way to average down uncertainty. Works for robotics navigation (next section); breaks down for reconstruction.
- **TSDF**: each new frame contributes a signed distance estimate. Weighted averaging smooths noise, and the zero-crossing extracts a clean surface. The tradeoff is that each cell stores two floats (value + weight) instead of a single byte, and the truncation band must be sized to the expected noise level.

Use TSDF when you need a clean dense mesh from a stream of noisy depth frames. Use binary occupancy when you need to answer "can I drive here?" quickly and compactly.

---

## robotics — probabilistic occupancy grids

### the outcome

A mobile robot navigating a warehouse needs to know: is this cell free? Can I plan a path through it? The robot cannot afford to build a dense mesh; it needs a compact, updatable map that it can query in real time and update as new sensor data arrives.

### how occupancy grids work

Elfes introduced the <em>occupancy grid</em> for mobile robot perception [[3]](#references): divide space into a regular grid of cells, and store in each cell the probability that the cell is occupied — P(occupied | sensor readings so far). Each new sensor reading (sonar, lidar, depth camera) updates the relevant cells via Bayes' rule.

In practice the update is done in <em>log-odds</em> form to avoid numerical instability and make the update additive rather than multiplicative:

```
l(n) ← l(n) + log( P(z|occupied) / P(z|free) )
```

where `l(n)` is the log-odds for cell n, and the sensor model `P(z|occupied) / P(z|free)` captures how likely the sensor reading z is under each hypothesis. The occupancy probability is recovered as:

```
P(occupied | z) = 1 - 1 / (1 + exp(l(n)))
```

Cells along a clear sensor ray get pushed toward "free"; cells at the measured surface get pushed toward "occupied." Over many sensor sweeps from different positions, the map converges to an accurate picture of the environment.

### OctoMap — 3D occupancy at scale

Elfes's original formulation was 2D (a floor plan). For 3D occupancy at the scale of building interiors, Hornung et al. built OctoMap [[4]](#references), which stores the probabilistic occupancy grid as an octree rather than a dense array.

An octree subdivides space recursively, like [octrees and SVOs](../storing/octrees-and-svo.md). The critical memory savings comes from the sparsity of real environments: most of the volume of a building is air, and an octree can represent large homogeneous regions as a single node rather than millions of individual cells. OctoMap can also represent the environment at multiple resolutions simultaneously — coarse for high-level path planning, fine for local collision checking — by querying at different octree depths.

The log-odds update rule is the same as Elfes's; the octree structure is purely a storage optimization. OctoMap also introduces clamping on the log-odds value to bound the time it takes a cell to change state when the environment changes (a moved shelf, an opened door).

#### occupancy grid vs TSDF for a robot

| | occupancy grid | TSDF |
|---|---|---|
| value per cell | P(occupied) | signed distance to surface |
| primary use | navigation, path planning | dense reconstruction, mesh extraction |
| noise handling | probabilistic averaging | weighted averaging |
| output | "can I go here?" | "what does the surface look like?" |
| memory per cell | ~1 byte (with octree compression) | 2 floats (value + weight) |
| updates | incremental per ray | incremental per frame |

Robotics systems that need both navigation and reconstruction sometimes maintain both: an OctoMap for global path planning and a local TSDF window for dense reconstruction near the robot.

---

## where the data goes next

Once a grid is populated from a real-world source, two paths are common:

**volume rendering** — medical CT and MRI data is usually not meshed; it is displayed directly as a volume by casting rays through the grid and compositing the density values. The Hounsfield calibration is what allows windowing to isolate tissue types. This is covered in [volume ray casting](../rendering/volume-ray-casting.md) and, for clinical workflows, in [scientific and medical volume rendering](../applications/scientific-and-medical-volume-rendering.md).

**VDB storage** — captured volumes from depth fusion or point cloud conversion are often large and sparse. Storing them as dense arrays is impractical; the standard industry format for sparse volumes is OpenVDB. See [OpenVDB and NanoVDB](../storing/openvdb-and-nanovdb.md) for how VDB's hierarchical structure fits these grids and what operations it supports efficiently.

---

## references

[1] Curless, B. and Levoy, M. (1996). "A Volumetric Method for Building Complex Models from Range Images." *Proceedings of SIGGRAPH '96*, pp. 303–312. [local PDF](../papers/curless-levoy-1996-tsdf-volumetric-range-images.pdf) · [source](https://graphics.stanford.edu/papers/volrange/)

[2] Newcombe, R.A., Izadi, S., Hilliges, O., Molyneaux, D., Kim, D., Davison, A.J., Kohli, P., Shotton, J., Hodges, S., and Fitzgibbon, A. (2011). "KinectFusion: Real-Time Dense Surface Mapping and Tracking." *Proceedings of ISMAR 2011*, pp. 127–136. DOI: 10.1109/ISMAR.2011.6092378. [local PDF](../papers/newcombe-2011-kinectfusion-dense-surface-mapping.pdf) · [source](https://www.microsoft.com/en-us/research/publication/kinectfusion-real-time-dense-surface-mapping-tracking/)

[3] Elfes, A. (1990). "Occupancy Grids: A Stochastic Spatial Representation for Active Robot Perception." *Proceedings of the Sixth Conference on Uncertainty in Artificial Intelligence*, pp. 60–70. [local PDF](../papers/elfes-1990-occupancy-grids-stochastic-spatial-representation.pdf) · [source](https://arxiv.org/abs/1304.1098)

[4] Hornung, A., Wurm, K.M., Bennewitz, M., Stachniss, C., and Burgard, W. (2013). "OctoMap: An Efficient Probabilistic 3D Mapping Framework Based on Octrees." *Autonomous Robots*, 34(3), pp. 189–206. DOI: 10.1007/s10514-012-9321-0. [local PDF](../papers/hornung-2013-octomap-probabilistic-3d-mapping.pdf) · [source](http://www.arminhornung.de/Research/pub/hornung13auro.pdf)
