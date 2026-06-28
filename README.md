# DICOM CT Preprocessing Pipeline — LIDC-IDRI

A complete preprocessing pipeline for clinical CT scans in DICOM format,
built on the LIDC-IDRI dataset — the original DICOM source that LUNA16 was derived from.

---

## Clinical Context

DICOM (Digital Imaging and Communications in Medicine) is the universal standard for
storing and transmitting medical images. Every CT scanner, MRI machine, and X-ray
system in clinical use produces DICOM files. Before any deep learning model can process
medical imaging data, this raw DICOM format must be decoded, validated, and normalised.

This pipeline covers the steps that happen before training — the part that is often
skipped in tutorials that hand you a clean numpy array. In real clinical or research
settings, this preprocessing is where most data engineering time is spent.

---

## What This Covers

Raw clinical CT data arrives in DICOM format. Before any model can use it, several
preprocessing steps are required. This notebook builds that pipeline from scratch,
including real problems encountered with the data and how they were solved.

| Step | What happens |
|---|---|
| Structure discovery | Navigate the unpredictable 3-level DICOM folder hierarchy |
| Header inspection | Read key metadata tags: spacing, position, rescale values |
| Slice sorting | Sort by physical z-position — not filename |
| Volume assembly | Stack 133–277 slices into a 3D numpy array |
| HU conversion | Convert raw integers → Hounsfield Units via slope/intercept |
| Resampling | Resample anisotropic voxels to 1mm isotropic spacing |
| Windowing | Clip to [−1000, 400] HU and normalise to [0, 1] |
| Batch metadata | Extract and compare spacing across 10 patients |
| Output | Save as `.npy` and `.mhd` — the format LUNA16 uses |

---

## Real Problems Documented

This notebook does not hide the issues encountered. Three problems appear in every
real clinical DICOM dataset — they are documented here with their fixes:

**1. Unpredictable folder names**
DICOM study and series folders are named from metadata (scan date, series UID),
not patient IDs. They look like `01-01-2000-NA-NA-30178`. Solution: always use
`rglob("*.dcm")` to find slices recursively regardless of folder names.

**2. Missing `ImagePositionPatient` tag**
2 of 135 slices were missing the standard z-position tag, causing an `AttributeError`.
Solution: fallback chain — try `ImagePositionPatient`, then `SliceLocation`,
then `InstanceNumber`. Drop slices that use the fallback to avoid geometry corruption.

**3. Non-CT files inside patient folders**
LIDC-IDRI folders contain annotation objects, structured reports, and presentation
states alongside CT slices — all with `.dcm` extension. Reading the first file found
returned wrong spacing values (−1.0mm, 0×0 dimensions).
Solution: filter by `Modality == "CT"` before reading any metadata.
This filter belongs in every DICOM project.

---

## Dataset

LIDC-IDRI (Lung Image Database Consortium and Image Database Resource Initiative)
200 patients, lung CT scans in DICOM format, with radiologist nodule annotations.

> Armato SG III, McLennan G, Bidaut L, et al. The Lung Image Database Consortium (LIDC)
> and Image Database Resource Initiative (IDRI): A completed reference database of lung
> nodules on CT scans. *Medical Physics* 38(2), 915–931 (2011).
> https://doi.org/10.1118/1.3528204

Kaggle: https://www.kaggle.com/datasets/washingtongold/lidcidri30

---

## Why Resampling Is Non-Negotiable

Across 10 patients, native scan spacing varied as follows:

| Property | Range observed |
|---|---|
| Slice thickness (z) | 1.25mm – 2.5mm |
| Pixel spacing (x/y) | 0.625mm – 0.879mm |
| Number of CT slices | 133 – 277 |

The same 5mm nodule would be 4 pixels tall in a 1.25mm scan and 2 pixels tall in a
2.5mm scan. Without resampling, the model must learn geometry correction alongside
pathology detection. Resampling to 1mm isotropic removes that variability entirely —
every voxel is a 1mm cube regardless of the scanner used.

---

## Results

After preprocessing, a representative patient volume (LIDC-IDRI-0001):

| Property | Before | After |
|---|---|---|
| Spacing | 0.703 × 0.703 × 2.500 mm | 1.0 × 1.0 × 1.0 mm |
| Shape | (133, 512, 512) | (332, 360, 360) |
| HU range | [−2048, 3071] | [0.0, 1.0] |

MHD output verified round-trip: shapes match, values match.

---

## Requirements

Python 3.10

```
pydicom==2.4.3
SimpleITK==2.2.1
numpy==1.24.0
pandas==2.0.0
matplotlib==3.7.0
```

## How to Run

1. Add the [LIDC-IDRI dataset](https://www.kaggle.com/datasets/washingtongold/lidcidri30) to your Kaggle notebook
2. Import `dicom_preprocessing_pipeline.ipynb` and run all cells top to bottom

---

## Connection to LUNA16

LUNA16 is LIDC-IDRI with DICOM already preprocessed and converted to MHD format.
This notebook performs the steps that happen *before* data like LUNA16 reaches you.
The `.mhd` files saved in the final section are in the exact same format as LUNA16
subset files — loading one with SimpleITK produces an identical result to this pipeline.
