<link rel="stylesheet" href="./css/globals.css">

# scientific and medical volume rendering

A radiologist loads a chest CT. On screen, a translucent 3D torso rotates slowly — the ribs glow white through soft pink lung tissue, a pulmonary artery snakes through the middle, and a small nodule sits beside a vessel, bright against the air around it. No scalpel has been lifted. No surface was ever extracted. The scanner recorded density at half a million points; the software is painting all of them at once, making each one as opaque or transparent as the tissue it represents. That is the whole trick.

The same idea runs through a fluid-dynamics simulation where a researcher needs to see shock waves forming inside a supersonic nozzle, or a structural biologist fitting an atomic model into a fuzzy blob of electron density from a cryo-EM experiment. The data is always a 3D grid of scalar values — CT density, simulation pressure, electron scattering — and the job is always to reveal the structure hidden inside it, not just on its surface.

This page is the applied companion to [volume ray casting](../rendering/volume-ray-casting.md), which derives the rendering integral. Here the focus is domain craft: what the data contains, how to expose it, and why medical and scientific rendering demands a different discipline from games.

---

## the coarse model — three things you need

Before any of the specifics, a beginner can hold a complete mental model in three parts:

1. **Data.** A 3D grid of numbers from a scanner or simulation — density, pressure, fluorescence, whatever the instrument measures. ([scanned and volume data](../generating/scanned-and-volume-data.md) covers how it arrives.)
2. **A function that decides what each number looks like.** You tell the renderer: "this density value should be translucent red, that one should be opaque white." That function — the <em>transfer function</em> — is the central creative and clinical act.
3. **A renderer that walks rays through the grid and accumulates color and opacity.** The math is in [volume ray casting](../rendering/volume-ray-casting.md). The point here is that every voxel along each ray contributes proportionally to the final pixel.

Those three pieces are enough to get the right answer. The rest of this page is about making it the *useful* answer.

---

## where the data comes from

CT and MRI scanners, like the ones that produce the volumes [scanned and volume data](../generating/scanned-and-volume-data.md) describes, deliver their data in <em>DICOM</em> (Digital Imaging and Communications in Medicine) format — the industry standard that wraps pixel arrays with rich metadata: patient orientation, slice thickness, acquisition parameters, and the physical scale of each voxel. A typical chest CT is 512×512 pixels per slice, 400–600 slices, stored as 16-bit integers representing x-ray attenuation.

Those integers are not arbitrary: CT uses the <em>Hounsfield Unit (HU)</em> scale, where water is 0 HU and air is −1000 HU. Different tissues fall at predictable positions:

| tissue | typical HU range |
|---|---|
| air (lung) | −1000 to −700 |
| fat | −120 to −50 |
| soft tissue / muscle | +20 to +80 |
| blood (unenhanced) | +30 to +50 |
| bone cortex | +700 to +3000 |

This predictability is what makes transfer functions tractable: instead of guessing where bone ends and soft tissue begins, you can look up where the boundary falls on the HU scale.

MRI does not use Hounsfield Units — its signal reflects proton density and relaxation times rather than attenuation, so absolute values are not standardized across scanners. Transfer functions for MRI often need to be retuned per scan rather than reused across patients.

Scientific simulation data (CFD pressure fields, astrophysics density cubes, cryo-EM electron scattering maps) arrives in domain-specific formats — HDF5, NetCDF, VTK XML, or MRC — but the voxel data model is the same: a 3D grid of scalars (or vectors) with spatial extent metadata. See [voxel data models](../foundations/voxel-data-models.md) for the storage layer.

---

## three rendering modes — described then named

Once you have a volume, you can turn it into an image in several fundamentally different ways. They are not competing implementations of the same thing; they answer different clinical or scientific questions.

### seeing the inside at every depth

The most general mode: cast a ray through the entire volume, accumulate the color and opacity of every voxel the ray passes through (weighted by each voxel's assigned opacity), and composite the result. Dense tissues appear opaque; lighter tissues stay translucent; the eye sees everything at once, depth-sorted by the accumulation. This is <em>direct volume rendering</em> (DVR). It is what the radiologist sees when looking at a translucent torso.

DVR asks the most of the transfer function because every voxel participates. Get the opacity wrong for soft tissue and it drowns the vessels behind it. The reward is that you see internal structure without committing to any particular surface.

### seeing the brightest thing along each ray

A simpler mode: for each ray, don't accumulate — just take the single brightest voxel the ray encounters and put it in the pixel. Bone and contrast-enhanced vessels — the brightest structures in a CT — pop out immediately against a black background. This is <em>maximum intensity projection</em> (MIP).

MIP is non-threshold-dependent (unlike iso-surface extraction, which requires you to pick a cutoff) and preserves attenuation values in the output. Radiologists reach for MIP specifically for vascular anatomy: CT angiography of coronary vessels, pulmonary emboli in pulmonary arteries, collateral vessels after stroke. It collapses 3D information into a 2D projection that reads like a conventional angiogram. The cost is ambiguity — superimposed bright structures merge, and you lose depth unless you rotate the volume or use thin-slab MIP (projecting through a limited depth range rather than the full volume).

### extracting a surface at a threshold

The third option is to pick a single density value — say +350 HU, which sits in the middle of bone — and extract the surface where the volume crosses that value, then render only that surface. This is iso-surface extraction. The standard algorithm for turning a voxel volume into that surface is marching cubes, covered in [marching cubes](../meshing/marching-cubes.md). The result is a polygon mesh that can be rendered with conventional rasterization at high frame rates.

Iso-surface rendering is used heavily for segmentation results (see below) and for 3D printing anatomical models. Its limitation is that it commits to a single threshold: soft tissues that blend continuously across HU values do not have a crisp boundary, and a threshold in the middle of the range creates an arbitrary, noise-sensitive surface.

#### when to reach for which

| question | mode |
|---|---|
| where is the tumor relative to the vessels? | DVR (translucency reveals both) |
| is the carotid artery narrowed? | MIP (vascular anatomy, angiographic look) |
| how large is the bone fragment? | iso-surface / marching cubes |
| what does the surface of the liver look like? | iso-surface after segmentation |
| where is the shock in this CFD run? | DVR (reveals gradients through the volume) |

---

## the transfer function — the central act

The transfer function is the thing you change when you want to see something different. It maps every scalar value in the volume to a color (R, G, B) and an opacity (α). Change the mapping, change what the image reveals.

### 1D transfer functions and windowing

The simplest form: a lookup table indexed by scalar value. Every voxel with value *v* gets color `C(v)` and opacity `α(v)`. In CT this is the basis of <em>windowing</em> (also called window/level): you choose a center HU value (the "level") and a range around it (the "width"), and the display maps that range to the full grayscale from black to white. Everything outside the window clips to black or white.

Radiologists use preset windows tuned to specific anatomy:

- **Lung window** — level ≈ −600 HU, width ≈ 1500 HU — spreads the low-attenuation air/lung range across the full grayscale; bone and soft tissue clip to white.
- **Brain window** — level ≈ +40 HU, width ≈ 80 HU — tightly zooms onto the soft-tissue range; tiny density differences between grey matter, white matter, and hemorrhage become visible.
- **Bone window** — level ≈ +400 HU, width ≈ 1800 HU — exposes the high-attenuation range; soft tissue clips to black.

These presets are saved as named configurations in clinical PACS (picture archiving and communication systems) and workstations. A radiologist switches between them in milliseconds. In 3D DVR the same logic applies: a 1D transfer function with a steep opacity ramp in the bone HU range will make bone opaque and leave soft tissue transparent.

The problem with 1D transfer functions is ambiguity at boundaries. Two different tissues often occupy overlapping HU ranges — partial-volume averaging smears the boundary across several voxels. The transition region is simultaneously "a bit of tissue A" and "a bit of tissue B." A 1D function cannot tell them apart; it only sees the mixed scalar value.

### 2D transfer functions — gradient-magnitude

Kindlmann and Durkin (1998) [1] observed that the boundary between two materials produces a characteristic signature: the scalar value is intermediate (ambiguous), but the *rate of change* — the gradient magnitude — is high. Pure interior voxels, far from any boundary, have low gradient magnitude. Boundary voxels have high gradient magnitude at intermediate scalar values.

Plot every voxel as a point in a 2D space — scalar value on the x-axis, gradient magnitude on the y-axis — and a recognizable pattern emerges: interior tissue clusters appear as horizontal blobs near the bottom (low gradient), and boundaries appear as arches rising up from the midpoints between two tissue clusters. Kindlmann and Durkin showed that placing opacity ramps along those arches, rather than as a function of scalar value alone, makes tissue boundaries visible without the ambiguity. A user can click on the arch in the 2D histogram and paint the boundary between, say, soft tissue and bone a specific color.

This <em>2D transfer function</em> indexed by (scalar value, gradient magnitude) is now standard in clinical volume rendering software. It separates "I'm at the surface of bone" from "I'm inside bone" — a distinction a 1D function cannot make.

### multidimensional transfer functions

Kniss, Kindlmann, and Hansen (2002) [2] extended this systematically. Their key insight: any second property of the volume — a second scalar channel (e.g., two co-registered MRI sequences), a second derivative of the density, or a different imaging modality — can form another axis of the transfer function. A 3D transfer function indexed by (T1 signal, T2 signal, gradient magnitude) can isolate tissue types that overlap in any single dimension but separate cleanly when two or three dimensions are combined simultaneously.

The authors paired multidimensional TFs with direct-manipulation widgets — 2D Gaussian blobs a user drags across the histogram to paint opacity — making a principled but previously academic technique into something an expert could interact with in real time.

Modern clinical systems use at least 2D (scalar + gradient) and sometimes 3D (two modalities + gradient) transfer functions. Research prototypes go further: texture features, curvature fields, and machine-learning classifiers have all been used as TF axes.

---

## segmentation and label volumes

Even the best transfer function cannot always separate overlapping tissue types. Bone and calcified plaques occupy the same HU range; a tumor inside liver may have nearly the same density as the surrounding parenchyma. The solution is to step outside the rendering pipeline and first classify every voxel by its anatomical identity — then render each label separately.

A <em>label volume</em> is a second grid, the same dimensions as the original scan, where each voxel carries an integer: 0 = background, 1 = liver, 2 = tumor, 3 = vessel, and so on. Once a label volume exists, you can:

- render only the liver in orange at 50% opacity while rendering the tumor in red at 100% opacity
- switch an organ invisible to look behind it
- measure volume, surface area, or distance to a margin
- export the surface of a labeled organ as a mesh for 3D printing or surgical planning

<em>Segmentation</em> is the process of producing that label volume. The spectrum of approaches:

- **Manual.** A radiologist draws outlines on slice-by-slice 2D views. Slow but precise. Still gold-standard for small structures.
- **Thresholding + region growing.** Pick a seed voxel inside a target organ, then expand outward as long as neighboring voxels fall within the expected HU range. Works well for liver and kidney, fails when the boundary is low-contrast.
- **Active contours / level-set methods.** A surface deforms under image forces until it snaps to a tissue boundary. Handles curved shapes and can incorporate shape priors.
- **Deep learning segmentation.** Convolutional networks (particularly U-Net variants) trained on annotated scans now achieve near-radiologist accuracy on many organs at inference speeds of seconds. Tools like TotalSegmentator can label 104 structures from a CT in under a minute on a desktop GPU.

In 3D Slicer, the workflow is: load DICOM → run automatic segmentation → review/correct on 2D slices → export label volume → apply per-label rendering in the 3D view.

---

## 2D slicing — multiplanar reformatting

Not every clinical question needs a 3D rendering. Most diagnostic reading is done on 2D slices because they preserve the full resolution of the original data and are fast to page through. But CT acquires data axially (cross-sections of the body from head to foot). Viewing the same dataset in the coronal plane (front-to-back) or sagittal plane (left-to-right) requires a resampling step.

That step — sampling the volume on any arbitrary plane and displaying the interpolated result as a 2D image — is <em>multiplanar reformatting</em> (MPR). Because modern multislice CT produces nearly isotropic voxels (equal resolution in all three axes), reformatted coronal and sagittal views are nearly as sharp as the original axial slices. Radiologists use them to trace a vessel, check a fracture line, or inspect anatomy that runs oblique to the scan plane — a curved MPR along the course of the aorta, for example.

MPR is computationally trivial: trilinear interpolation at each point on the target plane. Its clinical value is that it makes the volume navigable without any rendering pipeline at all.

---

## scientific use cases — beyond the clinic

The same rendering stack serves scientific domains outside medicine. The data formats change; the core problem — expose hidden structure in a dense 3D scalar field — does not.

- **Computational fluid dynamics (CFD).** A simulation of airflow over a wing produces pressure, velocity, and vorticity at every grid point. Volume rendering of pressure reveals shock surfaces; volume rendering of vorticity magnitude shows where turbulence concentrates. Transfer functions here are tuned by the scientist to the dynamic range of the simulation, not to HU presets.
- **Astrophysics.** Stellar formation simulations and galaxy surveys produce density and temperature fields at scales spanning many orders of magnitude. Log-scale transfer functions with opacity ramps on the dense filaments reveal cosmic structure. ParaView running on a supercomputer handles volumes that would not fit in the memory of any workstation.
- **Cryo-electron microscopy (cryo-EM).** A cryo-EM density map is a 3D grid of electron scattering potential, typically at 2–4 Å resolution for near-atomic work. The map is noisy. Volume rendering at an iso-surface contour level (chosen interactively to match the expected protein volume) reveals the molecular envelope; scientists then fit atomic models into the density. Tools like UCSF ChimeraX combine molecular visualization with volume rendering in the same scene.

In all these cases, the rendering problem is the same as in medicine: the structure of interest is buried inside a continuous scalar field, and the transfer function is the tool for extracting it. What differs is how the domain expert knows where the signal is.

---

## tooling — what real systems look like

Three open-source platforms dominate both clinical research and scientific visualization. They share VTK as a rendering engine, which means the underlying volume ray casting is consistent; they differ in audience and workflow.

### 3D Slicer

Built for medical image analysis. Loads DICOM natively, integrates segmentation tools (manual, thresholding, deep-learning via extensions), provides MPR views alongside a 3D volume-rendering pane, and has a MRML scene graph that keeps images, transforms, and models in one document. Used for surgical planning, radiotherapy target delineation, and medical research. The volume rendering module lets you pick preset transfer functions (CT-Bone, CT-Chest, CT-Abdomen) or design custom ones with a visual editor.

### ParaView

Built for large-scale scientific data. Handles structured and unstructured grids from CFD, climate models, and astrophysics simulations. Scales from a laptop to a distributed-memory supercomputer cluster using MPI. Transfer functions are tunable with the same histogram-based editor. Less oriented toward DICOM; dominant in the scientific simulation community.

### VTK (Visualization Toolkit)

The library both Slicer and ParaView are built on. If you need to integrate volume rendering into a custom application — a surgical guidance system, a research pipeline, a PACS viewer extension — you use VTK directly. It provides the ray casting renderer, the transfer function objects, the MPR resampler, and the marching-cubes iso-surface extractor all as programmable components.

GPU acceleration is table stakes in all three. The volume ray casting loop (casting a ray per pixel, sampling the volume, looking up the transfer function, accumulating opacity) maps naturally to a fragment shader or compute shader. Modern clinical CT volumes (512×512×600) render interactively at 30+ fps on a mid-range workstation GPU.

---

## medical vs. games — a different contract

Games rendering optimizes for visual impact; it is legitimate to invent detail, add specular highlights that don't correspond to physical materials, or soften shadows for aesthetic effect. The viewer knows it is fiction.

Medical volume rendering operates under a different contract: **invent nothing**. A radiologist measuring a nodule's diameter needs that measurement to correspond to physical reality. An intensity value that appears on screen needs to reflect the actual HU value at that anatomical location, not a tone-mapped approximation. A surface generated by marching cubes for surgical planning needs to match the tissue boundary in the scan, not smooth over it.

This has concrete technical implications:

- Transfer functions must be reproducible: the same preset applied to the same scan must produce the same image on every workstation.
- Rendering must be deterministic: stochastic techniques (like path tracing with noise) are not acceptable for primary diagnosis without validation.
- The original data — the 16-bit HU values — must be preserved and accessible at all times, independent of the rendering. The visualization is a view of the data, not the data itself.
- New rendering techniques (cinematic rendering, AI-enhanced visualization) must pass clinical validation before use in diagnosis; they may be powerful but they are not yet approved as primary diagnostic tools.

This constraint is why games-derived real-time rendering techniques enter clinical systems slowly even when the image quality improvement is obvious. The question is never just "does it look better?" but "does it preserve diagnostic information faithfully, and how do we know?"

---

## references

[1] Kindlmann, G. and Durkin, J. W. (1998). "Semi-Automatic Generation of Transfer Functions for Direct Volume Rendering." *Proceedings of the 1998 IEEE Symposium on Volume Visualization*, pp. 79–86. DOI: 10.1145/288126.288167. [local PDF](../papers/kindlmann-durkin-1998-semi-automatic-transfer-functions.pdf) · [source](https://people.cs.uchicago.edu/~glk/pubs/pdf/Kindlmann-SemiAutomaticTransferFunctions-VV-1998.pdf)

[2] Kniss, J., Kindlmann, G., and Hansen, C. D. (2002). "Multidimensional Transfer Functions for Interactive Volume Rendering." *IEEE Transactions on Visualization and Computer Graphics*, 8(3), 270–285. DOI: 10.1109/TVCG.2002.1021579. [local PDF](../papers/kniss-kindlmann-hansen-2002-multidimensional-transfer-functions.pdf) · [source](http://people.cs.uchicago.edu/~glk/pubs/pdf/Kniss-MultidimensionalTransferFunctions-TVCG-2002.pdf)

[3] Levoy, M. (1988). "Display of Surfaces from Volume Data." *IEEE Computer Graphics and Applications*, 8(3), 29–37. DOI: 10.1109/38.511. [local PDF](../papers/levoy-1988-display-surfaces-volume-data.pdf) · [source](https://graphics.stanford.edu/papers/volume-cga88/volume.pdf)

[4] Max, N. (1995). "Optical Models for Direct Volume Rendering." *IEEE Transactions on Visualization and Computer Graphics*, 1(2), 99–108. DOI: 10.1109/2945.468400. [local PDF](../papers/max-1995-optical-models-direct-volume-rendering.pdf) · [source](https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf)

[5] Prokop, M. (1997). "Use of Maximum Intensity Projections in CT Angiography: A Basic Review." *RadioGraphics*, 17(2), 433–451. DOI: 10.1148/radiographics.17.2.9084083. [source](https://pubs.rsna.org/doi/10.1148/radiographics.17.2.9084083)

[6] Salvolini, L., Bichi Secchi, E., Costarelli, L., and De Nicola, M. (2000). "Clinical Applications of 2D and 3D CT Imaging of the Airways." *European Journal of Radiology*, 34(1), 9–25. (MPR and airways: clinical translation overview.)

[7] Lorensen, W. E. and Cline, H. E. (1987). "Marching Cubes: A High Resolution 3D Surface Construction Algorithm." *ACM SIGGRAPH Computer Graphics*, 21(4), 163–169. DOI: 10.1145/37402.37422. (Foundational iso-surface algorithm; implementation covered in [marching cubes](../meshing/marching-cubes.md).)

[8] Fedorov, A., Beichel, R., Kalpathy-Cramer, J., et al. (2012). "3D Slicer as an Image Computing Platform for the Quantitative Imaging Network." *Magnetic Resonance Imaging*, 30(9), 1323–1341. DOI: 10.1016/j.mri.2012.05.001. [source](https://pmc.ncbi.nlm.nih.gov/articles/PMC3466397/)

[9] Preim, B. and Botha, C. (2013). *Visual Computing for Medicine: Theory, Algorithms, and Applications* (2nd ed.). Morgan Kaufmann. (Comprehensive treatment of DVR, TF design, segmentation, and clinical workflows.)
