<link rel="stylesheet" href="./css/globals.css">

# sparse voxel DAGs

Rendering whole buildings, cities, or billion-polygon scenes at interactive frame rates requires geometry data that fits in GPU memory — typically a few gigabytes at most. An SVO of a large scene can easily exceed that limit by an order of magnitude, even with empty space pruned away. The solution explored here is to find and merge the duplicate structure that almost every real scene contains, collapsing a tree into a graph so many parents share a single copy of the same subtree. The resulting structure is called a <em>sparse voxel DAG (SVDAG)</em>, and it is the approach that has made billion-to-trillion voxel scenes tractable on consumer hardware.

If you are new to the SVO, read [octrees and sparse voxel octrees](./octrees-and-svo.md) first — this page builds directly on it. The underlying motivation for all of this is covered in [the storage problem](./the-storage-problem.md).

---

## the key observation — an SVO is full of duplicates

An SVO prunes empty space by only keeping nodes that contain geometry. What it does not do is notice when two different subtrees contain exactly the same pattern of voxels. In a typical scene, that happens constantly:

- **Structural repetition.** Bricks, tiles, bolts, rivets, windows, leaves — architecture and nature are built from repeated motifs at every scale.
- **Empty space encodes identically.** A node with all eight children empty looks the same whether it sits in the sky above a building or deep inside a solid wall. An SVO stores thousands of copies of this "all-empty" node.
- **Symmetry.** A left wall and a right wall, a staircase going up and one going down — their subtrees are often identical or near-identical.

In a benchmark scene of the Unreal Engine's EpicCitadel voxelized to 128K³ resolution (roughly 19 billion occupied voxels), the SVO needed about 5.1 GB of GPU memory. The SVDAG representation of the same data fits in 945 MB — a roughly 5× reduction, achieved by sharing repeated subtrees rather than storing each one separately [1].

---

## from tree to graph — how sharing works

An octree is a tree: every node has exactly one parent. The insight behind the SVDAG is to relax that constraint. If two nodes are identical — same occupancy pattern, same children — they can be merged into a single node that multiple parents point to. The resulting structure is no longer a tree; it is a <em>directed acyclic graph</em> (DAG), where edges point from parent to child with no cycles, but a single node can be reached from many parents.

A sketch of what this looks like:

```
SVO (tree):            SVDAG (graph):
  A                      A
 / \                    / \
B   C      →           B   C
|   |                   \ /
D   D                    D   ← one node, two parents
```

In this toy example, the SVDAG stores D once instead of twice. In a real scene with millions of repeated leaf patterns and thousands of repeated internal clusters, the savings compound level by level through the tree.

The crucial property: a ray traversing the SVDAG follows exactly the same descent logic as an SVO. At each node it reads the 8-bit child mask, computes which child the ray enters, and follows the pointer. The fact that the pointer may be shared with other nodes is invisible to the traversal. No decompression step is needed; the compressed form is the traversal form [1].

---

## building a DAG — bottom-up deduplication

The standard construction algorithm works bottom-up, one level at a time, starting at the leaf nodes.

1. **Build the SVO first.** Start from the full voxelized scene (or stream it Morton-order) and construct a standard octree.
2. **Process leaves.** A leaf in a binary-occupancy SVO is just a 64-bit child mask (which of the 64 sub-voxels in a 4³ block are filled). Hash each distinct mask. Two leaves with the same hash are the same node; keep only one copy and update all parent pointers to reference it.
3. **Move up one level.** Now process the level above the leaves. Each node at this level is defined by its 8 child-pointers plus a child mask. Hash the tuple of pointers. Two nodes that reference the same set of children — now potentially already-merged shared nodes — are identical and can be merged.
4. **Repeat until the root.** Merge each level in turn. Because merging at level L can cause formerly different nodes at level L+1 to become identical (they now reference the same merged children), the savings compound upward.

The hash table lookup at each level is the entire mechanism. Two nodes are structurally identical if and only if their hash-of-children matches. No tree traversal or subtree comparison is needed; the hash makes identification O(1) per node [1].

The construction is done offline. The output is a flat array of nodes (with pointers replaced by array indices) that fits on the GPU. Building this structure from scratch at runtime is not feasible, which directly shapes when SVDAGs are and are not the right tool.

---

## compression in practice

The compression SVDAG achieves depends heavily on how much repetition the scene contains.

| structure | EpicCitadel 128K³ (19 billion voxels) |
|---|---|
| dense grid | tens of terabytes (impractical) |
| SVO | ~5.1 GB (pointer-free ideal) |
| SVDAG | ~945 MB |
| SSVDAG (with symmetry, see below) | ~86 MB at 0.123 bits/voxel [2] |

These are not outliers. Across tested scenes in the original paper, the number of nodes is reduced by **one to three orders of magnitude** compared to an SVO. Scenes with high structural regularity (architecture) compress far more than chaotic terrain or organic shapes [1].

---

## extensions — getting more out of structural similarity

### symmetry-aware DAGs (SSVDAG)

The plain SVDAG only merges subtrees that are bit-for-bit identical. But a left wall and its mirror image are nearly identical — only the child ordering is flipped along one axis. Villanueva, Marton, and Gobbetti's <em>symmetry-aware sparse voxel DAG (SSVDAG)</em> extends merging to subtrees that are identical under plane reflections along the main grid axes [2].

The encoding adds three bits to each child pointer: one bit per axis, indicating whether the child should be read with that axis mirrored before interpreting its occupancy. A node that would have been stored separately as the mirror of another node now shares the same underlying node, with the mirroring flag set. The compression is real: the same EpicCitadel scene that takes 945 MB as a plain SVDAG drops to about 86 MB as an SSVDAG, at 0.123 bits per voxel — roughly another 11× reduction [2].

The 2016 i3D paper introduced the approach; the 2017 JCGT journal extension fills in the theory and benchmark suite [2, 3].

### storing colors and other attributes

Geometry-only SVDAGs share nodes when the occupancy pattern matches. Colors break this immediately: a red brick and a blue brick have the same shape but different colors, so they can no longer share a node. Naively adding color to the SVDAG undoes most of the compression.

The approach from Dolonius, Sintorn, Kämpe, and Assarsson [4] decouples geometry from color entirely:

- The geometry DAG is built and compressed as normal — no change to the structural sharing.
- Each leaf in the DAG gets a pointer into a **separate 1D array of colors**, one entry per voxel on the surface.
- That 1D color array is then reshaped into a 2D image using a **space-filling curve** (which preserves spatial locality, so nearby voxels end up near each other in the image).
- The 2D image is compressed using standard GPU texture compression — BC7 or ASTC — which hardware decodes for free in a shader.

The geometry DAG stays fully shared. The color layer pays only for the surface voxels that actually have different colors, not for every node in the tree. Combined with BC7/ASTC, this achieves roughly 3× additional compression on the color data with very little perceptual loss [4]. The paper won Best Paper at I3D 2017.

The broader lesson from this line of work: adding any per-voxel attribute that varies independently of geometry (normals, material IDs, emissivity) forces a similar decoupling strategy. The geometry graph can stay compressed; attributes need their own representation layered on top. See [compression techniques](../optimization/compression-techniques.md) for where this fits in the broader landscape.

---

## the defining tradeoff — compression vs. mutability

The properties that make SVDAGs so effective at compression are exactly what make them difficult to edit.

**Why editing is hard:**

- Every node may be shared by many parents scattered across the tree. Changing one voxel means finding every parent that references the affected subtree, potentially splitting a shared node into a unique copy for the modified path and a still-shared copy for everything else.
- The savings come from merging. An edit that makes a previously-shared subtree unique unmerges it — potentially cascading up through multiple levels.
- The structure lives on the GPU as a flat array built offline. There is no natural place to insert or delete nodes at runtime.

**HashDAG — making edits practical.** Careil, Billeter, and Eisemann (2020) showed that SVDAGs can be embedded in a hash table (the HashDAG), where the hash of a node's children is its address. Edits reconstruct only the affected path bottom-up, writing new nodes into the hash table, and old unique nodes are reclaimed. This enables interactive large-scale edits — carving, filling, painting — on compressed geometry without full decompression [5].

The practical summary of when to reach for an SVDAG:

| situation | fit |
|---|---|
| static scene, ray-traced rendering | excellent — small memory, fast traversal |
| large architectural or repeated-motif geometry | excellent — high structural regularity |
| simulation that writes to voxels per-frame | poor — plain SVDAG is immutable |
| interactive editing (carve/fill) | possible with HashDAG, with overhead |
| colored/attributed geometry | workable with decoupled color layer [4] |

For scenes where the geometry changes frequently, a [dense grid or chunked structure](./dense-grids-and-chunks.md) or a [hash grid](./hash-grids-and-bricks.md) is usually a better starting point. For static high-resolution geometry where rendering quality and memory are the primary concerns, the SVDAG is hard to beat. How this sits relative to SVOs and VDB is covered in [choosing a voxel store](./choosing-a-voxel-store.md).

---

## svdag vs svo vs dense — at a glance

| | dense grid | SVO | SVDAG | SSVDAG |
|---|---|---|---|---|
| empty space | wastes all of it | pruned | pruned | pruned |
| repeated structure | duplicated | duplicated | **merged** | **merged + mirrored** |
| typical memory (large scene) | impractical | GB | hundreds of MB | tens of MB |
| editable at runtime | yes (write a cell) | partial | no (without HashDAG) | no (without HashDAG) |
| ray traversal | fast DDA | moderate | moderate (no decompression) | moderate |
| build time | trivial | fast | offline (minutes) | offline (slower) |
| per-voxel attributes | direct | direct | requires decoupling [4] | requires decoupling |

For the rendering side — how a ray descends this structure to produce an image — see [sparse voxel octree ray tracing](../rendering/sparse-voxel-octree-raytracing.md).

---

## references

[1] Kämpe, V., Sintorn, E., and Assarsson, U. (2013). "High resolution sparse voxel DAGs." *ACM Transactions on Graphics*, 32(4), Article 124. DOI: 10.1145/2461912.2462024. [local PDF](../papers/kampe-sintorn-assarsson-2013-high-resolution-sparse-voxel-dags.pdf) · [source](https://www.cse.chalmers.se/~uffe/HighResolutionSparseVoxelDAGs.pdf)

[2] Villanueva, A. J., Marton, F., and Gobbetti, E. (2016). "SSVDAGs: symmetry-aware sparse voxel DAGs." *Proceedings of the 20th ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (i3D 2016)*, pp. 7–14. DOI: 10.1145/2856400.2856420. [local PDF](../papers/villanueva-marton-gobbetti-2016-ssvdag-symmetry-aware-sparse-voxel-dags.pdf) · [source](https://www.crs4.it/vic/data/papers/i3d2016-symmetry-dags.pdf)

[3] Villanueva, A. J., Marton, F., and Gobbetti, E. (2017). "Symmetry-aware sparse voxel DAGs (SSVDAGs) for compression-domain tracing of high-resolution geometric scenes." *Journal of Computer Graphics Techniques*, 6(2), pp. 1–30. ISSN: 2331-7418. [local PDF](../papers/villanueva-marton-gobbetti-2017-ssvdag-jcgt.pdf) · [source](https://www.crs4.it/vic/data/papers/jcgt2017-ssvdags.pdf)

[4] Dolonius, D., Sintorn, E., Kämpe, V., and Assarsson, U. (2017). "Compressing color data for voxelized surface geometry." *Proceedings of the 21st ACM SIGGRAPH Symposium on Interactive 3D Graphics and Games (i3D 2017)* (Best Paper). Extended version in *IEEE Transactions on Visualization and Computer Graphics*, 25(2), pp. 1270–1282 (2019). DOI: 10.1109/TVCG.2017.2741480 (journal). I3D DOI: 10.1145/3023368.3023381. [local PDF](../papers/dolonius-sintorn-kampe-assarsson-2017-compressing-color-voxelized-surface-geometry.pdf) · [source](https://www.cse.chalmers.se/~uffe/dolonius2017i3d.pdf)

[5] Careil, V., Billeter, M., and Eisemann, E. (2020). "Interactively modifying compressed sparse voxel representations." *Computer Graphics Forum*, 39(2), pp. 111–119. DOI: 10.1111/cgf.13916.
