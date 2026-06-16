<link rel="stylesheet" href="./css/globals.css">

# machine learning on voxels

A self-driving car needs to know, right now, whether the space two metres ahead is occupied by a pedestrian, a parked truck, or nothing at all. A robotics lab wants a neural network that can identify a coffee mug from a depth scan and estimate its pose. A generative model needs to complete a partial 3D scan of a room from a single photograph. All of these are tasks in 3D perception and reconstruction — and the central question is: what data structure should a neural network work on?

The answer has changed several times in a decade, and the voxel grid has played a different role at each stage. Understanding why tells you something real about what neural networks need and what 3D data costs.

---

## the coarse model: why any representation at all

A neural network that classifies images works on a 2D grid — a rectangular array of pixels, arranged by row and column. The convolution operator slides a small kernel across that grid, reads neighboring values, and produces a new grid of features. The architecture generalizes trivially to three dimensions: replace the 2D grid with a 3D grid, replace the 2D kernel with a 3D kernel, and you have a 3D convolutional neural network.

The catch is that a voxel grid large enough to be useful is expensive. Doubling the resolution in every axis multiplies the cell count by eight. A 64³ grid holds 262 thousand cells; a 256³ grid holds 16 million. Most of those cells, for most objects in most scenes, are empty.

That cubic memory cost is the wall every voxel-based learning system eventually hits, and the history of 3D deep learning is largely a sequence of techniques for hitting that wall more gently — or for going around it entirely.

---

## why the grid first: regular structure is a gift

### the tensor shortcut

A 3D occupancy grid is just a 3D array of numbers. Each cell stores a value — typically 0 for empty, 1 for occupied, or a continuous density — and the spatial position of each cell is implicit in its array index, exactly as in a 2D image. That regularity means:

- standard convolution generalizes directly: a 2D conv kernel becomes a 3D one, with no new math
- the entire deep learning infrastructure — backpropagation, GPU memory management, batching — works without modification
- the network can learn spatial patterns (edges, corners, cylinders) that are translation-invariant in all three axes

This is the gift that made voxel grids the first choice for 3D deep learning.

### VoxNet and 3D ShapeNets: the proof of concept

The two papers that established this approach appeared within months of each other.

**3D ShapeNets** (Wu et al., CVPR 2015) [1] represented each CAD model as a probability distribution over a 30³ voxel grid and trained a Convolutional Deep Belief Network on it. The result was the first large-scale demonstration that a deep network could learn 3D shape features directly from occupancy voxels — classifying shapes, completing partial scans from a depth camera, and planning the next-best view to disambiguate an object. It also produced ModelNet, the 3D shape dataset that benchmarked the field for years.

**VoxNet** (Maturana & Scherer, IROS 2015) [2] took the same core insight and applied it to real sensor data: LIDAR point clouds from robots and autonomous vehicles. VoxNet voxelized each LIDAR sweep into a 32³ <em>occupancy grid</em> — a 3D tensor whose cells recorded the probability of occupancy inferred from sensor returns — and fed that tensor to a small 3D CNN with two convolutional layers. Despite the tiny architecture, it ran at real-time speeds and demonstrated that 3D convolution was practical for robotics perception.

Both nets were bottlenecked at the same place: 32³ or 30³ was about as far as GPU memory of that era could reach. Any attempt to raise resolution ran directly into the O(n³) wall.

---

## the wall and the fix: sparsity is the key

### why dense convolution wastes compute

Real 3D data — a LIDAR scan, a depth reconstruction, a segmented CT slice — is overwhelmingly empty. A scene from an autonomous vehicle might have 95% of its voxels holding air. A dense 3D convolution reads all of those empty cells, multiplies them by kernel weights, and writes outputs for them — the vast majority of the work produces only zeros.

The fix is to skip the empty cells entirely: only compute convolutions where data actually exists. Because occupied voxels live on the surfaces and boundaries of objects, they form a thin shell — a submanifold of the full 3D space — that is far smaller than the enclosing grid.

### submanifold sparse convolution

Benjamin Graham and Laurens van der Maaten identified the core issue with naive sparse convolution: each layer of a standard sparse conv expands the set of active sites outward, because the kernel reaches into the neighborhood of each occupied cell. After several layers, the "active" region has grown to fill most of the volume, erasing the sparsity advantage entirely.

Their solution, <em>submanifold sparse convolution</em> [3], constrains output activations to exist only at positions that were occupied in the input. The convolution reads from the neighborhood as usual, but it only writes to locations that were already active. The sparsity pattern is preserved through every layer. This keeps memory and compute proportional to the number of occupied voxels rather than the grid volume, allowing much higher resolutions than dense convolution could reach.

### OctNet: hierarchical sparsity

A complementary approach uses an octree to focus resolution where the scene is complex and coarsen it where it is empty. **OctNet** (Riegler et al., CVPR 2017) [4] introduced a hybrid data structure that stacks a shallow array of octrees, each subdividing only to the depth that local content warrants. Operations on the network are defined on the tree structure, so compute scales with occupied content rather than bounding volume. OctNet demonstrated 3D convolution on grids as fine as 256³ — eight times finer per axis than VoxNet — while staying within the memory budget that 32³ dense grids had required.

The sparse storage structures that make this possible at serving time are covered in [hash grids and bricks](../storing/hash-grids-and-bricks.md).

### MinkowskiNet: sparse tensors as a general framework

**MinkowskiNet** (Choy, Gwak & Savarese, CVPR 2019) [5] generalized sparse convolution into a complete autodifferentiation framework for arbitrary-dimensional sparse tensors. The key object is a sparse tensor: a set of coordinates paired with feature vectors, with no dense backing array. Convolutions, pooling, and skip connections all operate on this representation.

This opened up a fourth dimension: time. By treating a sequence of 3D scans as a 4D sparse tensor — space plus the scan index as a temporal axis — MinkowskiNet could apply the same sparse convolution framework to video-like sequences of point clouds, and achieved state-of-the-art results on large-scale 3D semantic segmentation benchmarks (ScanNet, S3DIS) that would have been unreachable with dense grids.

---

## the representation contest

A voxel grid is not the only way to give a neural network a 3D input. Three other representations now compete with it, each making different tradeoffs. The choice is the lesson.

### point clouds — PointNet and descendants

Rather than voxelizing the sensor output, why not process the raw points directly? A LIDAR sweep is already a set of 3D coordinates; converting it to a voxel grid throws away sub-cell precision and pays the cost of a full grid allocation.

**PointNet** (Qi et al., CVPR 2017) [6] showed this was possible. The key insight: any function on an unordered set can be approximated by individually transforming each point and then applying a symmetric aggregation (like max-pooling) across the set. Max-pooling is order-invariant, so the network doesn't care what order the points arrive in — the <em>permutation invariance</em> problem that had made point clouds hard to learn from is handled structurally, not by preprocessing.

PointNet processes each point independently and globally aggregates; it is fast and memory-efficient, but it struggles with local geometric detail because each point never sees its neighbors until the global pool. Follow-up work (PointNet++, PointConv, KPConv) added local neighborhood aggregation, but at the cost of irregular memory access patterns that make point-based methods harder to batch and harder to fuse with other data sources.

See [voxels vs other representations](../foundations/voxels-vs-other-representations.md) for the full comparison of how these representations trade storage against access cost.

### neural / implicit fields — Occupancy Networks and NeRF

Both PointNet and voxel grids represent geometry as explicit data: either a set of points or a dense array. A third family of approaches represents geometry *implicitly*, as a function that answers "is this point in space inside the object?" without ever storing the shape as discrete samples.

**Occupancy Networks** (Mescheder et al., CVPR 2019) [7] trained a neural network to predict the occupancy probability at any continuous 3D coordinate, conditioned on a shape encoding. Because the function is continuous and never discretized into a grid, the effective resolution is theoretically unlimited — you query the network at whatever density you need at inference time. A single network can represent a highly detailed surface that would require a 512³ or higher voxel grid to capture at equivalent quality, within a few megabytes of network weights.

**NeRF** (Mildenhall et al., ECCV 2020) [8] extended the implicit approach to full scene appearance: a neural network maps a 3D position and a viewing direction to a color and a volume density. Rendering a novel view means casting rays through the scene and integrating the network outputs along each ray — the same volume rendering integral used in scientific visualization (covered in [scientific and medical volume rendering](./scientific-and-medical-volume-rendering.md)), but with the density field stored as network weights rather than a voxel grid. NeRF produced photo-realistic view synthesis from sparse camera images, a task where explicit representations had long fallen short.

Where voxels re-enter the picture: the original NeRF is slow. Querying the MLP thousands of times per pixel per image is the bottleneck. Instant-NGP (Müller et al., SIGGRAPH 2022) [9] replaced the large MLP with a multi-resolution hash grid — a voxel-like structure at several scales that stores small learned feature vectors at each cell — reducing training from hours to seconds. The hash grid is exactly the kind of multi-resolution spatial structure described in [hash grids and bricks](../storing/hash-grids-and-bricks.md), now used not to store geometry directly but to store features that a small neural network reads.

### 3D Gaussian splatting — points that render

**3D Gaussian Splatting** (Kerbl et al., SIGGRAPH 2023) [10] represents a scene as a set of 3D Gaussians — ellipsoidal blobs in space, each with a position, scale, orientation, opacity, and color. Rendering projects these Gaussians onto the image plane and alpha-blends them in depth order using a GPU rasterizer, achieving real-time frame rates. There is no grid and no ray-marching network query: the representation is explicit and point-based, and the rendering path is fast because it uses the same tile-based rasterizer that handles traditional geometry.

Gaussian splatting achieves NeRF-quality results at interactive speeds, and as of 2024 has become the dominant choice for novel-view synthesis from images. The rendering mechanics are covered in [splatting and point rendering](../rendering/splatting-and-point-rendering.md).

### the contrast table

| representation | memory cost | resolution | convolution | sensor fusion | edit / query |
|---|---|---|---|---|---|
| dense voxel grid | O(n³) | low (≤128³ typically) | trivial 3D conv | natural | O(1) lookup |
| sparse voxel / submanifold | O(occupied) | high | sparse conv | natural | O(1) with hash |
| point cloud | O(points) | sensor resolution | irregular, hard to batch | direct from LIDAR | nearest-neighbor search |
| implicit / neural field | O(weights) | continuous (unlimited) | not applicable | indirect | slow (MLP query per point) |
| 3D Gaussian splatting | O(Gaussians) | high | not applicable | indirect | per-Gaussian update |

---

## where voxels still win

The implicit and point-based representations have taken the lead on view synthesis and shape reconstruction. Voxel grids — dense or sparse — retain real advantages in specific contexts:

- **regular convolution**: 3D CNNs on voxel grids use highly optimized GPU kernels that are hard to match with irregular point-based networks. For tasks like volumetric classification or dense scene completion, 3D sparse convolution remains competitive or dominant.
- **sensor fusion**: a voxel grid is the natural common reference frame for fusing multiple sensors. A camera image and a LIDAR sweep arrive in different coordinate systems with different densities; projecting both into a shared voxel grid gives a uniform structure that a convolutional network can process without special handling of mismatched formats. This is central to autonomous driving perception stacks.
- **occupancy prediction**: predicting which voxels around an ego vehicle are occupied (and by what class) is now a standard task in autonomous driving — the CVPR 2023 3D Occupancy Prediction challenge formalized it. The output is inherently a voxel grid, and the inputs (multi-camera images, LIDAR) are fused into one. The representation that suits the task and the sensor data determines the architecture.
- **fast spatial lookup**: knowing whether a point in space is occupied is an O(1) operation on a voxel grid. Physics simulations, path planners, and collision detectors need this constantly and at predictable latency. Neural fields answer the same question only after an MLP forward pass.
- **scanned volume data**: CT and MRI data already arrive as voxel grids. Applying 3D convolution directly to medical volume data, without conversion, is natural. See [scanned and volume data](../generating/scanned-and-volume-data.md) for how this data originates and what it carries.

The applications that live at the intersection of robotics, autonomous driving, and medical imaging — where data arrives as sensor grids, where spatial lookup matters, and where the output is itself a 3D spatial structure — are where voxel-based learning remains the right tool.

---

## where voxels lose

- **resolution vs. memory**: a 256³ grid at float32 needs 64 MB per channel. A scene reconstruction that needs millimeter detail at room scale is out of reach for dense grids.
- **fine surface detail**: the grid quantizes everything to cell size. Thin structures, sharp edges, and sub-cell features are lost or aliased.
- **view synthesis**: for tasks where the goal is photo-realistic rendering from novel viewpoints, Gaussian splatting and NeRF-family methods produce sharper, more faithful results than voxel grids can support at tractable resolutions.
- **generative modeling**: generating diverse, detailed 3D shapes at high resolution is more natural in implicit or point-based spaces, where the output does not carry an O(n³) memory penalty.

The full tradeoff picture — not just for learning but for every use of voxels — is in [voxels vs other representations](../foundations/voxels-vs-other-representations.md). For the breadth of applications beyond perception and learning, see [voxels beyond games](./voxels-beyond-games.md).

---

## references

[1] Wu, Z., Song, S., Khosla, A., Yu, F., Zhang, L., Tang, X., and Xiao, J. (2015). "3D ShapeNets: A Deep Representation for Volumetric Shapes." *Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition (CVPR)*, 1912–1920. [local PDF](../papers/wu-2015-3d-shapenets-deep-representation-volumetric-shapes.pdf) · [source](https://arxiv.org/abs/1406.5670)

[2] Maturana, D. and Scherer, S. (2015). "VoxNet: A 3D Convolutional Neural Network for Real-Time Object Recognition." *Proceedings of the IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS)*, 922–928. [local PDF](../papers/maturana-2015-voxnet-3d-cnn-object-recognition.pdf) · [source](https://www.ri.cmu.edu/pub_files/2015/9/voxnet_maturana_scherer_iros15.pdf)

[3] Graham, B. and van der Maaten, L. (2017). "Submanifold Sparse Convolutional Networks." arXiv:1706.01307. [local PDF](../papers/graham-2017-submanifold-sparse-convolutional-networks.pdf) · [source](https://arxiv.org/abs/1706.01307)

[4] Riegler, G., Ulusoy, A. O., and Geiger, A. (2017). "OctNet: Learning Deep 3D Representations at High Resolutions." *Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition (CVPR)*, 3577–3586. [local PDF](../papers/riegler-2017-octnet-deep-3d-representations-high-resolutions.pdf) · [source](https://arxiv.org/abs/1611.05009)

[5] Choy, C., Gwak, J., and Savarese, S. (2019). "4D Spatio-Temporal ConvNets: Minkowski Convolutional Neural Networks." *Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition (CVPR)*, 3075–3084. arXiv:1904.08755. [local PDF](../papers/choy-2019-minkowskinet-4d-spatio-temporal-convnets.pdf) · [source](https://arxiv.org/abs/1904.08755)

[6] Qi, C. R., Su, H., Mo, K., and Guibas, L. J. (2017). "PointNet: Deep Learning on Point Sets for 3D Classification and Segmentation." *Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition (CVPR)*, 652–660. [local PDF](../papers/qi-2017-pointnet-deep-learning-point-sets.pdf) · [source](https://arxiv.org/abs/1612.00593)

[7] Mescheder, L., Oechsle, M., Niemeyer, M., Nowozin, S., and Geiger, A. (2019). "Occupancy Networks: Learning 3D Reconstruction in Function Space." *Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition (CVPR)*, 4460–4470. [local PDF](../papers/mescheder-2019-occupancy-networks-3d-reconstruction-function-space.pdf) · [source](https://arxiv.org/abs/1812.03828)

[8] Mildenhall, B., Srinivasan, P. P., Tancik, M., Barron, J. T., Ramamoorthi, R., and Ng, R. (2020). "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis." *Proceedings of the European Conference on Computer Vision (ECCV)*, 405–421. arXiv:2003.08934. [local PDF](../papers/mildenhall-2020-nerf-neural-radiance-fields-view-synthesis.pdf) · [source](https://arxiv.org/abs/2003.08934)

[9] Müller, T., Evans, A., Schied, C., and Keller, A. (2022). "Instant Neural Graphics Primitives with a Multiresolution Hash Encoding." *ACM Transactions on Graphics (SIGGRAPH)*, 41(4), Article 102. DOI: 10.1145/3528223.3530127. [source](https://nvlabs.github.io/instant-ngp/)

[10] Kerbl, B., Kopanas, G., Leimkühler, T., and Drettakis, G. (2023). "3D Gaussian Splatting for Real-Time Radiance Field Rendering." *ACM Transactions on Graphics (SIGGRAPH)*, 42(4), Article 139. arXiv:2308.04079. [local PDF](../papers/kerbl-2023-3d-gaussian-splatting.pdf) · [source](https://arxiv.org/abs/2308.04079)
