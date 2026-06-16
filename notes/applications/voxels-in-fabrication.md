<link rel="stylesheet" href="./css/globals.css">

# voxels in fabrication

Imagine a prosthetic grip that is rigid at its mounting bracket and gradually melts into a soft, skin-like cushion at the contact surface — no seam, no glue joint, a continuous transition through the interior of the part. Or a lightweight structural bracket whose core is a fine lattice of air and polymer, tuned cell by cell to be stiff where load concentrates and open where it doesn't. Or a lifelike replica of a coral specimen printed at one-to-one scale with each voxel carrying the measured color and translucency of the real tissue.

None of these can be described by a surface mesh. They require something that can hold data at every point through the interior of a solid — which is exactly what a voxel grid is for.

This page covers how voxel representations align with the physical process of additive manufacturing, why they unlock capabilities that the standard STL pipeline cannot express, and what the fabrication workflow looks like from design to printed part.

---

## why the printer and the voxel grid are the same thing

Additive manufacturing builds objects by depositing material in small increments — a layer of photopolymer cured by UV light, a pass of inkjet print heads laying down droplets, a line of extruded filament. Every one of these processes divides the build volume into discrete spatial units and decides, unit by unit, what to put there.

That is the definition of a voxel grid.

A PolyJet printer, for instance, jets photopolymer droplets onto a build tray at a resolution of 600 × 300 dots per inch in XY and roughly 14–27 µm per layer in Z. Each droplet is one voxel. Before the printer can fire a single nozzle, it needs to know the material assignment for every one of those voxels across every layer. The instruction set the printer actually consumes is a stack of per-layer bitmap images — one pixel per voxel, one value per material — which is precisely a voxel grid sliced into 2D sheets [3].

When a design is expressed as a voxel model, this translation is direct: the voxel grid becomes the print instruction. When a design is expressed as a surface mesh (STL), a conversion pipeline must reconstruct the interior, guess at material assignments, and then discretize everything at the printer's native resolution. The information that was never in the file cannot be recovered in that step.

---

## what a surface mesh cannot say

A [triangle mesh](../foundations/voxels-vs-other-representations.md) describes the boundary of an object — where the surface is. The interior is implicit: it is assumed to be a single uniform material, solid all the way through. The STL format, which is still the dominant exchange format for 3D printing, encodes only triangle positions and face normals. It has no field for material, no concept of interior structure, and no way to say "this region is stiff here and soft there."

This is fine for a single-material part with a known fill pattern. It is not fine when the design intent is:

- a grip whose stiffness transitions from 0.5 MPa (soft rubber) to 50 MPa (rigid polymer) over 15 mm of depth
- a bracket whose interior is 30% dense lattice rather than solid polymer
- a multi-material object where color and opacity vary continuously through the interior, not just on the painted surface

For these, the surface mesh simply has no room to carry the information. The material distribution through the volume must be described volumetrically — which means a voxel model (or its close cousin, an [implicit SDF/CSG model](../generating/sdf-and-csg-modeling.md), which can be converted to a voxel grid at print time).

---

## multi-material and functionally graded printing

When a printer can jet two or more base materials simultaneously — depositing different resins from different nozzles in the same pass — it can blend materials at the voxel level. A voxel assigned to material A and one assigned to material B, interleaved in a fine pattern, produce a composite whose bulk properties are intermediate between A and B. Change the ratio of A-to-B voxels and you change the effective material.

This is called <em>multi-material printing</em>, and it is what makes the soft-to-hard grip possible: you specify a spatial function that maps position in the part to a blend ratio, and the printer realizes that function droplet by droplet.

When the blend ratio changes continuously and smoothly through the volume — so there is no sharp boundary between hard and soft, only a gradient — the result is called a <em>functionally graded material (FGM)</em>. The term comes from materials science, where it describes composites whose composition changes through their thickness. In 3D printing, an FGM is realized by assigning each voxel a precise material mix ratio rather than a binary choice.

Hiller and Lipson's foundational work on <em>digital materials</em> — voxel-level compositions of two or more base resins — showed that mixing ratio, voxel geometry, and precision all independently control bulk properties including elastic modulus, density, coefficient of thermal expansion, and failure mode [1, 2]. This established the theoretical grounding for treating voxel composition as a continuous design variable rather than a discrete manufacturing constraint. Doubrovski et al. (2015) applied the same principle practically, designing a customized prosthetic socket where stiffness was mapped per-voxel using bitmap printing — the printer's native resolution was used directly, with no slicing or path planning required beyond generating a per-layer image stack [10].

---

## the voxel-to-printer workflow

### from model to bitmap stack

The practical fabrication pipeline has three stages.

**1. Design in a volumetric representation.** Either model directly as a voxel grid, assigning material IDs or blend ratios cell by cell; or design using signed distance functions and CSG operations (see [SDF and CSG modeling](../generating/sdf-and-csg-modeling.md)) and convert to voxels at the end. For scanned objects — CT data, MRI, microscopy — the data already arrives as a voxel volume; see [mesh voxelization](../generating/mesh-voxelization.md) for the conversion from mesh inputs.

**2. Slice into per-layer bitmaps.** The voxel volume is <em>sliced</em> — cut into horizontal planes at the printer's layer thickness — producing one 2D image per layer. Each pixel carries a material code: which resin to jet at that XY position on that layer. The Stratasys J750 accepts one PNG file per layer, with pixel colors mapped to up to six loaded materials [3]. At 14 µm layers, a 1 cm tall part requires about 714 bitmaps; a 10 cm part requires about 7,140.

**3. Print.** The printer steps through the stack layer by layer, reading each bitmap and firing nozzles accordingly.

### the streaming problem

At native printer resolution, the full resolved voxel grid for a moderately sized part can reach hundreds of gigabytes. Storing the entire pre-computed array is often impractical.

Vidimče et al.'s OpenFab system (2013) addresses this with a streaming pipeline inspired by GPU rendering [4]. Just as a fragment shader computes a pixel's color on demand from its position, an OpenFab *fablet* computes each voxel's material composition on demand from its spatial coordinates — evaluating a procedural material function rather than reading from a pre-stored array. The pipeline processes the build volume slice by slice, keeping only one layer in memory at a time. This makes it practical to design parts whose material varies by a continuous mathematical function — a spatial gradient, a noise-driven microstructure, an analytically defined FGM — without ever materializing the full grid.

### dithering: turning a gradient into discrete droplets

A real printer can only jet discrete materials — it cannot jet "30% rigid + 70% flexible" as a single droplet. The bridge between a continuous material gradient and a finite palette of base resins is <em>dithering</em>.

Dithering is borrowed directly from 2D halftoning: just as a laser printer approximates a gray tone by varying the density of black dots on white paper, a multi-material printer approximates a blend ratio by varying the proportion of A-droplets to B-droplets within a neighborhood of voxels. At a scale larger than the individual droplet, the mechanics perceive the blend rather than the individual components.

3D dithering extends this to volume: each voxel's material is assigned probabilistically based on the target blend ratio at that position, stepping through X, Y, then Z. The technique has measurable structural benefits beyond cosmetics: Bibb et al. (2023) showed that 3D random dithering over a 20 mm gradient length improves interfacial toughness by 40.8% compared to a sharp material boundary, because the interlocked voxel pattern mechanically reinforces the transition zone rather than leaving a stress-concentrating seam [5].

Bader et al. (2018) demonstrated the pipeline end-to-end for data physicalization: CT and MRI volumetric datasets, confocal microscopy stacks, and protein fiber tractography were each converted to per-voxel material assignments and printed on PolyJet hardware, with binary raster files generated per material per layer [6]. The result is a physical object whose interior faithfully encodes the original volumetric data — possible only because the representation and the print instruction format are the same thing.

---

## microstructures, lattices, and metamaterials

Per-voxel control does more than enable material gradients. It enables *architectural* control of the interior — designing not which material fills each voxel, but what geometric pattern of voids and solids fills each region.

A lattice infill replaces solid material with a repeating open-cell structure of polymer struts and air, reducing mass while preserving load-bearing capacity. At larger scales, this is ordinary slicer infill. At the scale of individual print droplets, where the cell geometry is designed with the wavelength of sound or the buckling length of thin columns in mind, the structure becomes a <em>microstructure</em> or <em>metamaterial</em> — a material whose bulk properties (acoustic, mechanical, electromagnetic) emerge from geometry rather than chemistry.

Voxel-level control is what makes designed microstructures printable. The design specifies a unit cell geometry and a spatial function mapping position to cell parameters: wall thickness, strut angle, local density. At each voxel, the appropriate geometry is instantiated. Bickel et al. (2010) demonstrated this for mechanical compliance: a multi-material printer was instructed voxel by voxel with layer arrangements optimized to produce a part with a user-specified nonlinear stress-strain curve — fabricating materials with designed deformation behavior that no single base resin could provide [7]. The design workflow relied on a volumetric material model rather than a surface mesh, because the target property lived entirely in the interior.

---

## format landscape

### why STL falls short, and what replaces it

STL was designed in 1987 for single-material stereolithography. It encodes triangles only. Sending a multi-material design as a set of STL files — one per material zone — forces every region to be a separate closed mesh, with no way to express gradients or shared boundaries. Interior structure is simply not representable.

3MF (3D Manufacturing Format) is the modern replacement: an XML-based archive that encodes geometry, color, material, texture, and build metadata in one file. Its Volumetric Extension (in pre-release as of 2026) adds explicit support for voxel grids and SDF fields, allowing a single file to carry a complete interior material specification. Liu et al. (2023) demonstrated a pipeline that reads 3MF surface color data and diffuses it inward through voxel fill to assign interior colors for full-color printing [8].

### the FAV format

In 2016, Fuji Xerox and Keio University proposed a purpose-built voxel exchange format: <em>FAV</em> (Fabricatable Voxel). Where STL encodes surface polygons, FAV breaks a model to the voxel level and allows each voxel to carry color, material identity, and connection strength to its neighbors — including internal structure data that polygon formats cannot hold [9].

FAV was standardized as Japanese Industrial Standard JIS B9442 in 2019. Version 1.1a allows each voxel to be subdivided into sub-units for finer attribute resolution. Its key design goal is round-trip fidelity: a FAV file travels from design tool to printer without an intermediate conversion that would strip interior information, which is exactly what polygon-based formats cannot guarantee.

---

## bioprinting: cells as voxels

The most material-specific case of per-voxel assignment is bioprinting, where the "materials" are living cells suspended in a hydrogel carrier. A bioprinter deposits cell-laden droplets layer by layer, with different cell types assigned to different spatial positions to replicate tissue architecture — hepatocytes in one zone, endothelial cells lining printed channels in another.

The design representation is a voxel grid whose values are cell-type identifiers rather than polymer codes. The fabrication process is structurally identical to multi-material polymer printing: each voxel position receives a specific material — here, a specific cell suspension — based on a pre-computed spatial assignment. The difference is that the "curing step" is biological: cells must adhere, proliferate, and self-organize after deposition, turning the printed geometry into living tissue over hours to days.

---

## closing the loop: representation determines what you can build

The choice of representation — surface mesh vs. voxel grid vs. implicit field — is not a file format question. It determines what design information can exist at all.

| | STL / triangle mesh | voxel grid | implicit / SDF |
|---|---|---|---|
| material per voxel | no | yes | via conversion |
| interior gradient (FGM) | no | yes | yes |
| microstructure / lattice | no (shell only) | yes | yes |
| slicing to bitmaps | requires reconstruction | direct | convert then slice |
| streaming at print resolution | not needed | memory-intensive | efficient (OpenFab [4]) |
| design-time editability | easy | heavy at full res. | compact, smooth |

Voxel models are the natural representation for additive manufacturing because they match the physical medium: depositing material cell by cell is building a voxel grid. The STL pipeline survives because most printed parts are still single-material, and for those the mesh→slice→infill workflow is fast and sufficient. As soon as interior material composition becomes part of the design intent, the surface shell must be replaced by a volumetric model — either an explicit voxel grid or an implicit field converted to one at print time.

This is the connection back to [voxels beyond games](./voxels-beyond-games.md): fabrication is the domain where voxels are not a rendering shortcut or a simulation convenience but the direct specification language of the machine itself.

For how voxel models compare to meshes and SDFs in more detail, see [voxels vs other representations](../foundations/voxels-vs-other-representations.md). For the storage structures behind large voxel volumes, see [voxel data models](../foundations/voxel-data-models.md). For generating voxel volumes from mesh inputs — the conversion direction that feeds many printing pipelines — see [mesh voxelization](../generating/mesh-voxelization.md).

---

## references

[1] Hiller, J. and Lipson, H. (2009). "Design and Analysis of Digital Materials for Physical 3D Voxel Printing." *Rapid Prototyping Journal*, 15(2), 137–149. DOI: 10.1108/13552540910943441. (Paywalled — establishes digital materials as voxel-level composition of base resins; shows composition controls elastic modulus, density, CTE, and failure mode.)

[2] Hiller, J. and Lipson, H. (2010). "Tunable Digital Material Properties for 3D Voxel Printers." *Rapid Prototyping Journal*, 16(4), 241–247. DOI: 10.1108/13552541011049252. (Paywalled — extends [1]: continuous tuning of bulk properties by varying voxel geometry and material precision.)

[3] Stratasys. (2021). "Guide to Voxel Printing" (GrabCAD Print support documentation). Describes the BMP/PNG per-layer bitmap workflow, 600×300 DPI XY resolution, and per-voxel material assignment on J750 hardware. [source](https://support.stratasys.com/en/Software/GrabCAD-Print/Tips-Guides-and-FAQs/Guide-to-Voxel-Printing)

[4] Vidimče, K., Wang, S.-P., Ragan-Kelley, J., and Matusik, W. (2013). "OpenFab: A Programmable Pipeline for Multi-Material Fabrication." *ACM Transactions on Graphics*, 32(4), Article 136 (SIGGRAPH 2013). DOI: 10.1145/2461912.2461993. [local PDF](../papers/vidimce-2013-openfab-multi-material-fabrication.pdf) · [source](https://vidimce.org/publications/openfab/)

[5] Bibb, R. et al. (2023). "Three-Dimension Dithering and Its Effect on the Interfacial Strength of Multi-Material and Emulated Multi-Material Additive Manufacturing Processes." *Additive Manufacturing*. DOI: 10.1016/j.addma.2023.103792. [source](https://repository.lboro.ac.uk/articles/journal_contribution/Three-dimension_dithering_and_its_effect_on_the_interfacial_strength_of_multi-material_and_emulated_multi-material_additive_manufacturing_processes/24534238)

[6] Bader, C., Kolb, D., Weaver, J. C., Sharma, S., Hosny, A., Costa, J., and Oxman, N. (2018). "Making Data Matter: Voxel Printing for the Digital Fabrication of Data across Scales and Domains." *Science Advances*, 4(5), eaas8652. DOI: 10.1126/sciadv.aas8652. [source](https://www.science.org/doi/10.1126/sciadv.aas8652)

[7] Bickel, B., Bächer, M., Otaduy, M. A., Lee, H. R., Pfister, H., Gross, M., and Matusik, W. (2010). "Design and Fabrication of Materials with Desired Deformation Behavior." *ACM Transactions on Graphics*, 29(4) (SIGGRAPH 2010). DOI: 10.1145/1778765.1778800. [source](https://cdfg.mit.edu/publications/design-and-fabrication-materials-desired-deformation-behavior)

[8] Liu, Y. et al. (2023). "Color Printing Based on 3MF: Color Diffusion from the Surface to the Interior of Voxel Model." *Engineering Reports*, e12623. DOI: 10.1002/eng2.12623. [source](https://onlinelibrary.wiley.com/doi/full/10.1002/eng2.12623)

[9] Fuji Xerox / FUJIFILM Business Innovation and Keio University. (2016). FAV (Fabricatable Voxel) 3D Data Format. Standardized as JIS B9442 (2019, rev. v1.1a). [source](https://www.fujifilm.com/fb/en/about/initiatives/technical/production/solution-service/fav)

[10] Doubrovski, E. L., Tsai, E. Y., Dikovsky, D., Geraedts, J. M. P., Herr, H., and Oxman, N. (2015). "Voxel-Based Fabrication through Material Property Mapping: A Design Method for Bitmap Printing." *Computer-Aided Design*, 60, 3–13. DOI: 10.1016/j.cad.2014.01.009. (Paywalled — demonstrates bitmap printing for a customized prosthetic socket; establishes the per-voxel stiffness-mapping workflow.) [source](https://www.sciencedirect.com/science/article/abs/pii/S0010448514001067)

[11] Wade, C., Beck, D., and MacCurdy, R. (2025). "Implicit Toolpath Generation for Functionally Graded Additive Manufacturing via Gradient-Informed Slicing." arXiv:2505.08093. [source](https://arxiv.org/abs/2505.08093)
