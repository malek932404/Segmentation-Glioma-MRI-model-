Brain Tumor Segmentation and Optimal Slice Selection Pipeline
1. Overview

This repository implements a complete pipeline for the analysis of multi-parametric brain MRI scans from the BraTS 2020 dataset. The system performs two core tasks:
•	3D consistent tumor segmentation using a 2D U-Net trained on axial slices, with robust normalization and patient-wise data splitting to prevent leakage.
•	Automatic identification of the optimal axial slice defined as the slice exhibiting the maximum predicted whole-tumor (WT) surface area to support efficient visual review and feature extraction for downstream tasks.

2. Data Requirements

	Input Format
All data must be preprocessed and stored in the following structure within the `content/data/` directory:
content/data/
├── volume_{ID}_slice_{IDX}.h5    (One file per axial slice)
├── BraTS20 Training Metadata.csv ( Slice level labels)

Each `.h5` file must contain two datasets:
- `image`: `(240, 240, 4)` — MRI modalities in order: `[FLAIR, T1, T1ce, T2]`
- `mask`: `(240, 240, 3)` — Binary tumor subregion masks: `[NCR/NET, ED, ET]`

3. Pipeline Architecture

The workflow is divided into three sequential stages, implemented in a single Jupyter notebook.

	Stage 1: Data Indexing and Patient Splitting
- Slice indexing: All `.h5` files are parsed and grouped by `volume_id`.
- Normalization: Robust per-volume z-scoring using 0.5th–99.5th intensity percentiles.
- Splitting: Patients are randomly assigned to train (70%), validation (15%), and test (15%) sets using a fixed seed (`1337`) to ensure reproducibility.
- Outputs:
  - `splits_patientwise.json`: Patient IDs per split.
  - `volume_stats.csv`: Per-patient statistics including ground-truth best slice (max WT area) and survival data (if available).

	Stage 2: 2D U-Net Segmentation Training
Model: Lightweight 2D U-Net with 4 input channels (MRI modalities) and 3 output channels (tumor subregions).
- Preprocessing: Per-slice robust z-normalization for training efficiency.
- Sampling: Optional weighted sampling to oversample tumor-containing slices (requires metadata CSV).
- Loss Function: Combined Binary Cross-Entropy + Soft Dice Loss per class.
- Training: Optimized with Adam (`lr=1e-3`), early stopping based on validation loss.
- Outputs:
  - `seg_all_slices_unet.pt`: Best PyTorch model checkpoint.
  - `unet_traced.pt`: TorchScript-compiled model for deployment.

	Stage 3: Optimal Slice Selection and Visualization
- Inference: Full-volume prediction using the trained U-Net.
- Best Slice Criterion: The axial slice with the maximum smoothed whole-tumor (WT) area, where  
  `WT = NCR/NET ∪ ED ∪ ET`.
- Smoothing: A 5-slice moving average is applied to the WT area curve to reduce noise.
- Visualization: For 10 random test patients:
  - Displays MRI background (highest-contrast channel).
  - Overlays ground truth and predicted segmentations.
  - Reports Dice scores per tumor subregion and for whole tumor.

4. Outputs and Artifacts

| File | Description |
|------|-------------|
| `splits_patientwise.json` | Reproducible patient-wise train/val/test splits |
| `volume_stats.csv` | Per-patient tumor metrics and clinical data |
| `seg_all_slices_unet.pt` | Trained U-Net weights (PyTorch) |
| `unet_traced.pt` | Optimized TorchScript model |
| Jupyter notebook outputs | Best-slice visualizations with Dice scores |



