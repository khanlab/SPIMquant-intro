---
marp: true
theme: default
paginate: true
header: "SPIMquant — Intro Tutorial & Demo"
footer: "Khan Lab | https://github.com/khanlab/SPIMquant"
style: |
  section {
    font-size: 1.4rem;
  }
  section.title {
    text-align: center;
    justify-content: center;
  }
  code {
    font-size: 0.9rem;
  }
  h1 { color: #2c5f8a; }
  h2 { color: #3a7abf; }
  a { color: #2c7fb8; }
---

<!-- _class: title -->

# SPIMquant

### Automated Quantitative Analysis of Lightsheet Microscopy Data

**Khan Lab**
https://github.com/khanlab/SPIMquant
https://spimquant.readthedocs.io

---

## What is SPIM?

**Selective Plane Illumination Microscopy** (Lightsheet)

- Whole-brain fluorescence imaging at cellular resolution
- Common applications:
  - Beta-amyloid / alpha-synuclein plaque mapping (AD/PD models)
  - Microglia density (Iba1)
  - Cholinergic neurons (ChAT)
  - Vasculature (CD31, Lectin)
- Results in **large, multi-resolution OME-Zarr datasets**

### Challenge

> How do we go from a terabyte-scale whole-brain image to **quantitative, region-specific statistics**?

---

## SPIMquant Overview

A **Snakemake / BIDS App** for automated quantification of lightsheet brain microscopy data.

**Key capabilities:**
- Deformable registration to a brain atlas template
- Atlas-based segmentation and quantification
- Multi-stain support (plaques, cells, vessels)
- Participant-level **and** group-level statistics

**Links:**
- 🔗 GitHub: https://github.com/khanlab/SPIMquant
- 📖 Docs: https://spimquant.readthedocs.io
- 🧰 Requires: Linux 64-bit, Pixi, ≥32 GB RAM

---

## Pipeline Overview

```
Raw SPIM data  →  SPIMprep  →  BIDS dataset (OME-Zarr)
                                      │
                              SPIMquant (participant)
                                      │
               ┌──────────────────────┼──────────────────────┐
               │                      │                       │
         Registration           Segmentation           Vessel seg
         (template)             (plaques/cells)         (CD31/Lectin)
               │                      │                       │
               └──────────────────────┴──────────────────────┘
                                      │
                               Quantification
                          (field fraction, counts,
                           density, region stats)
                                      │
                        SPIMquant (group statistics)
```

---

## SPIMprep — Prerequisite

**SPIMprep** converts raw acquisition data into a BIDS-compatible dataset with OME-Zarr images.

**Accepts:**
- Stitched Imaris (`.ims`) files
- Multi-tile TIFF stacks
- Other common lightsheet formats

**Produces:**
- BIDS dataset with `sub-*/micr/*_SPIM.ome.zarr`
- Multi-resolution image pyramid (levels 0–5+)
- Metadata in `dataset_description.json`, `participants.tsv`

> SPIMprep ensures SPIMquant always receives well-structured, validated input data.

🔗 https://github.com/khanlab/spimprep

---

## Registration Pipeline

Aligns each SPIM image to a common brain atlas template.

### What it does

1. **Import**: Convert OME-Zarr → NIfTI at a downsampled level
2. **Masking**: Generate brain mask with Gaussian Mixture Model (GMM)
3. **Rigid registration**: Coarse alignment at downsampled resolution (default: level 5)
4. **Deformable registration**: High-quality nonlinear alignment (greedy / ANTs)
5. **QC report**: HTML visual inspection report

```bash
# Registration-only dry run (no segmentation, no vessels)
pixi run spimquant /bids /out participant \
  --no-segmentation --no-vessels -np
```

---

## Registration Pipeline — Outputs

| File | Description |
|------|-------------|
| `xfm/*_from-SPIM_to-{template}_*.mat` | Forward transform (SPIM → template) |
| `xfm/*_regqc.html` | HTML registration QC report |
| `xfm/*_regqc.png` | QC snapshot image |
| `micr/*_space-{template}_SPIM.nii.gz` | SPIM in template space |
| `parc/*_from-{template}_dseg.nii.gz` | Atlas parcellation in native SPIM space |

### Supported Templates

| Template | Species | Notes |
|----------|---------|-------|
| ABAv3 | Mouse | Allen Brain Atlas v3 (default) |
| DSURQE | Mouse | Dorr atlas, 40 µm MRI |
| gubra | Mouse | Gubra atlas |
| MBMv3 | Marmoset | Primate studies |
| turone | Mouse | Turone atlas |

---

## Registration Pipeline — MRI Co-registration

Optional: register a corresponding MRI to SPIM space (and vice versa).

```bash
pixi run spimquant /bids /out participant \
  --register-to-mri \
  --template-mri DSURQE
```

**Output:**
- `anat/*_space-{template}_via-SPIM_desc-preproc_T2w.nii.gz`
  MRI registered to template space via the SPIM → template transform

> Enables direct comparison of structural MRI features with microscopy signals.

---

## Quantification Pipeline — Segmentation

Segment the signal of interest in the registered SPIM image.

### Intensity Correction

- **N4 bias field correction** (default) or Gaussian-based correction
- Corrects spatial intensity non-uniformity before thresholding

### Segmentation Methods

| Method | Description |
|--------|-------------|
| `otsu+k3i2` | Multi-Otsu thresholding (3 classes, index 2) — **default** |
| `threshold` | Manual intensity threshold |

```bash
pixi run spimquant /bids /out participant \
  --seg-method otsu+k3i2 \
  --correction-method n4
```

> Better methods in active development (e.g., self-supervised deep learning segmentation)

---

## Quantification Pipeline — Field Fraction

**Field fraction** = proportion of voxels above the segmentation threshold per region.

### Computation

1. Binary segmentation mask (foreground = 1)
2. Downsample by averaging → values range 0–1 per voxel
3. Transform to template space using saved registration

### Outputs

| File | Description |
|------|-------------|
| `seg/*_fieldfrac.nii.gz` | Field fraction in native SPIM space |
| `seg/*_space-{template}_fieldfrac.nii.gz` | Field fraction in template space |
| `tabular/*_seg-{roi}_segstats.tsv` | Field fraction aggregated by atlas ROI |
| `seg/*_seg-{roi}_fieldfrac.nii.gz` | Heatmap volumes per atlas parcellation |

---

## Quantification Pipeline — Instance-Based Metrics

For counting **individual objects** (plaques, cells) rather than continuous signal.

### Method

- Connected-component labeling run on **overlapping chunks** (chunk-based parallelism via Dask)
- Objects touching chunk boundaries are handled via overlap regions
- Only instances with **centroids inside the chunk** are retained → no double-counting

### Available Metrics

| Metric | Description |
|--------|-------------|
| `count` | Number of instances per atlas region |
| `density` | Instances per unit regional volume |
| `nvoxels` | Size distribution of instances |
| `volume` | Total volume of instances per region |

---

## Quantification Pipeline — Vessel Analysis

Specialized pipeline for **vascular segmentation** (CD31, Lectin stains).

### Method

- **VesselFM** deep-learning model for vessel detection
- **Signed distance transform** from vessel surfaces
  - Used to relate plaques/cells with nearby vasculature (e.g., CAA analysis)

### Outputs

| File | Description |
|------|-------------|
| `vessels/*_fieldfrac.nii.gz` | Vessel field fraction (volume density) |
| `vessels/*_density.nii.gz` | Vessel count density |

> Next-generation vessel graph analysis is in active development.

---

## Running the Pipeline — Basic Usage

```bash
# Install (one-time)
curl -fsSL https://pixi.sh/install.sh | bash
git clone https://github.com/khanlab/SPIMquant.git
cd SPIMquant && pixi install

# Validate (dry run)
pixi run spimquant /path/to/bids /path/to/output participant -np

# Full participant-level analysis
pixi run spimquant /path/to/bids /path/to/output participant --cores all

# Group-level statistics
pixi run spimquant /path/to/bids /path/to/output group \
  --contrast-column treatment \
  --contrast-values control drug \
  --cores all
```

---

## Running the Pipeline — Key CLI Options

### Processing Control

```
--registration-level N     Downsampling level for registration (default: 5)
--segmentation-level N     Downsampling level for segmentation (default: 0)
--no-segmentation          Skip segmentation & quantification
--no-vessels               Skip vessel analysis
--sloppy                   Low-quality params for quick testing
```

### Template & Atlas

```
--template {ABAv3|DSURQE|gubra|MBMv3|turone}
--atlas-segs [SEGS ...]    Atlas parcellations to use
```

### Subject Selection

```
--participant-label 01 02 03
```

---

## Running the Pipeline — Cluster & Cloud

### SLURM Cluster

```bash
pixi run spimquant /bids /out participant \
  --jobs 50 \
  --executor slurm
  --default-resources
```



### Useful Debug Options

```
-n / --dry-run         Show what would run without executing
-R rule_name           Re-run a specific rule
--until rule_name      Run only up to a specific rule
--report               Generate an HTML workflow report
--notemp               Keep all intermediate files
```

---

## Demo — Subject-Level Outputs

### What to look at for a single subject

1. **Registration QC report** — `xfm/*_regqc.html`
   - Check alignment of SPIM to template brain
   - Look for gross misregistration or mask failures

2. **Field fraction heatmap** — `seg/*_seg-roi22_fieldfrac.nii.gz`
   - Colour-coded plaque density per brain region in template space

3. **TSV stats table** — `tabular/*_seg-roi22_segstats.tsv`
   - Per-region numbers ready for downstream analysis

---

## Demo — Batch / Group Analysis

### After processing a cohort

```
output/
└── group/
    ├── *_groupstats.tsv         # t-stat, p-value, Cohen's d per region
    ├── *_groupstats.png         # Heatmap visualization
    ├── *_groupstats.nii         # 3-D volumetric statistical maps
    ├── *_groupavgsegstats.tsv   # Group-averaged stats per region
    └── *_groupavg.{tstat|pval|cohensd}.nii.gz
```

### Typical workflow

1. Run participant level for all subjects
2. Run group level with `--contrast-column` / `--contrast-values`
3. Explore region-level TSV files in Python / R
4. Visualise volumetric stat maps in ITK-SNAP or FSLeyes

---

## Summary

| Stage | Tool | Key Output |
|-------|------|-----------|
| Raw → BIDS | SPIMprep | `sub-*/micr/*_SPIM.ome.zarr` |
| Masking | SPIMquant | Brain mask |
| Registration | SPIMquant | Transforms + QC report |
| Segmentation | SPIMquant | Binary mask, field fraction |
| Quantification | SPIMquant | TSVs, heatmap NIfTIs |
| Vessel analysis | SPIMquant | Vessel density maps |
| Group stats | SPIMquant | t-stat / p-val / Cohen's d maps |

**SPIMquant turns terabyte-scale whole-brain microscopy into publication-ready quantitative results.**

---

<!-- _class: title -->

## Questions?

📖 Documentation: https://spimquant.readthedocs.io/en/latest/
💻 GitHub: https://github.com/khanlab/SPIMquant
📧 Contact: Ali Khan — alik@robarts.ca

---

## Appendix — Stain Support

| Stain | Target | Pipeline |
|-------|--------|----------|
| PI / YOPRO / AutoF | Nuclear / autofluorescence | Registration channel |
| Abeta / BetaAmyloid | Beta-amyloid plaques (AD) | Segmentation |
| AlphaSynuclein | Lewy bodies (PD) | Segmentation |
| Iba1 | Microglia | Segmentation |
| ChAT | Cholinergic neurons | Segmentation |
| CD31 / Lectin | Blood vessels | Vessel pipeline |

---

## Appendix — Output Directory Structure

```
output/
├── sub-{id}/
│   ├── micr/        # Registered SPIM images (NIfTI + OME-Zarr)
│   ├── xfm/         # Transforms + registration QC reports
│   ├── seg/         # Segmentation masks + field fraction maps
│   ├── parc/        # Atlas parcellations in native space
│   ├── vessels/     # Vessel density maps
│   ├── tabular/     # Per-region statistics (TSV)
│   └── anat/        # MRI (if --register-to-mri used)
└── group/
    ├── *_groupstats.tsv
    ├── *_groupstats.png
    └── *_groupavg.*.nii.gz
```

---

## Appendix — Installation Details

```bash
# 1. Install Pixi
curl -fsSL https://pixi.sh/install.sh | bash

# 2. Clone SPIMquant
git clone https://github.com/khanlab/SPIMquant.git
cd SPIMquant

# 3. Install all dependencies (creates isolated environment)
pixi install

# 4. Verify
pixi run spimquant --help
```

**System requirements:**
- Linux x86-64
- ≥32 GB RAM (64+ GB recommended for large datasets)
- SSD storage for optimal I/O performance
- No GPU required (CPU-only pipeline)
