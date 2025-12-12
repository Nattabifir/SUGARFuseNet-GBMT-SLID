# SUGARFuseNet / GBMT‑SLID

SUGARFuseNet: Diffusion‑Driven Domain Adaptation and Bimodal Bitemporal Fusion for Advancing Global Landslide Segmentation on the novel GBMT‑SLID dataset.

Repository for the paper:
SUGARFuseNet: Diffusion‑Driven Domain Adaptation and Bimodal Bitemporal Fusion for Advancing Global Landslide Segmentation on novel GBMT‑SLID dataset.

Author / maintainer: Franck-Emani  
Contact: franckemani@yahoo.ca

Contents
- Overview
- Data preprocessing
- Diffusion-based data enhancement (cDMT + DDIM)
- Models (SUGARFuseNet + baselines)
- Training & validation
- Inference & evaluation
- Results & reproducibility
- Citation & license
- Contributing

Overview
--------
This repository contains code, scripts and instructions to reproduce the experiments described in our manuscript. We provide:
- GBMT‑SLID dataset preparation and organization (pre/post Sentinel‑2 + Copernicus DEM)
- A diffusion-driven data enhancement strategy comprising:
  - condition Diffusion Model Translator (cDMT) for domain adaptation between pre- and post-event optical images (guided by change maps), and
  - DDIM-based augmentation to synthesize additional labelled samples and mitigate class/regional imbalance.
- The SUGARFuseNet segmentation architecture implementing:
  - Shared Gated Paired Attention (SGPA)
  - Bimodal Bitemporal Information Fusion Module (BBIFM)
  - Feature Aggregation (FA) multi-scale learning
  - Unified Features Upsampling Module (UFUM)
- Training pipelines, baseline implementations, and inference scripts for evaluation on unseen regions.

GBMT‑SLID dataset (summary)
---------------------------
GBMT‑SLID (Global Bimodal Bitemporal Sentinel Landslide Inventories Dataset)
- Coverage: 29 geographically diverse regions worldwide.
- Modalities per sample:
  - Pre-event optical image (Sentinel‑2)
  - Post-event optical image (Sentinel‑2)
  - Topographic image (Copernicus DEM-derived features: elevation, slope, aspect — precomputed)
- Labels: pixel-wise landslide masks (binary: landslide / non-landslide)
- Typical tile size: (user-defined; in our experiments we used 256×256 patches) — adapt as needed.
- Recommended structure (example):
  ```
  GBMT-SLID/
  ├── region_<id>/
  │   ├── pre/             # Sentinel‑2 pre-event tiles (TIF or numpy)
  │   ├── post/            # Sentinel‑2 post-event tiles
  │   ├── dem/             # Copernicus DEM / slope / aspect maps
  │   ├── mask/            # ground-truth masks (0/1)
  │   └── meta.json        # acquisition dates, bbox, coords, change-map if available
  ```

Data preprocessing
------------------
Key preprocessing steps we used (implemented in Data preprocessing/):
- Co-registration: ensure subpixel alignment of pre/post optical and DEM rasters.
- Cloud masking & cloud/shadow filtering (Sentinel-2 QA60 / Sentinel Hub masks or Fmask).
- Radiometric normalization: atmospheric-correction approach (e.g., Sen2Cor outputs) or histogram matching when necessary.
- Resampling: bring DEM/topographic products to optical resolution (~10 m for Sentinel‑2) using bilinear/cubic for continuous features and nearest for masks.
- Change map generation: compute a coarse change indicator (e.g., differenced band indices or band-wise change detection) to guide cDMT domain adaptation and sample selection.
- Patch extraction & balancing: extract patches with stride and perform class-balanced selection for training/val/test splits.

Diffusion-based data enhancement (Data Enhancement Strategy)
-----------------------------------------------------------
Our two-stage Data Enhancement Strategy (DES):
1) condition Diffusion Model Translator (cDMT)
   - Purpose: harmonize pre-event appearance to the post-event style (or vice-versa) only in unchanged areas, preserving true changes.
   - Input: pre-event image + change map mask (unchanged area conditioning)
   - Output: translated pre-image aligned with post imagery distribution in unchanged regions.
   - Use-case: reduces domain gap between pre and post pairs for paired model training.

2) DDIM-based augmentation
   - Purpose: generate additional realistic samples (optical & DEM) to alleviate regional imbalance and enrich landcover variability.
   - Operates after cDMT to produce multiple plausible augmented post-event samples for a given scene.
   - Parameters to tune: sampling steps, guidance scale, seed control.

Scripts (examples):
- Diffusion based data enhancement/
  - cDMT_train.py          # train conditional diffusion translator
  - cDMT_translate.py      # translate pre->post using change mask
  - ddim_augment.py        # DDIM sampling wrapper for augmentation
  - augmentation_catalog.json # metadata for augmented samples

Model zoo
---------
- SUGARFuseNet (primary)
  - Dual-branch encoder handling bimodal-bitemporal inputs (pre/post optical + DEM)
  - Shared Gated Paired Attention (SGPA) block for cross-branch contextual exchange
  - Bimodal Bitemporal Information Fusion Module (BBIFM) in the encoder
  - Feature Aggregation (FA) for multi-scale learning
  - Unified Features Upsampling Module (UFUM) in decoder for boundary recovery

- Benchmarks / Baselines (included implementations or reproduction wrappers)
  - Semantic segmentation: TransUNet, SegFormer, CMFNet
  - Change detection: ChangeFormer, SiAUNet, Siamese Swin-UNet
  - Landslide-mapping multimodal: SCDUNet, BBUNet

Models/ directory contains:
- sugarfuse/               # implementation of SUGARFuseNet
- baselines/               # baseline implementations and wrappers
- losses/                  # Dice, Focal, BCE, combined losses
- utils/                   # model utils, weight init, schedulers

Training & validation
---------------------
Requirements (example)
- Python >= 3.8
- Tensorflow >= 2.10 (CUDA-enabled)
- numpy, rasterio, GDAL, tqdm, scikit-learn, scikit-image

Install (suggested)
```
# recommended: create conda env
conda create -n sugarfuse python=3.9
conda activate sugarfuse
pip install -r requirements.txt
```

Typical training commands (replace paths and hyperparams as needed):
```
# Train SUGARFuseNet
python Models/sugarfuse/train.py \
  --data_root /path/to/GBMT-SLID \
  --train_list data/train_list.txt \
  --val_list data/val_list.txt \
  --batch_size 8 \
  --epochs 100 \
  --lr 1e-4 \
  --save_dir ./exp/sugarfuse_run1 \
  --use_dem True \
  --pretrained_backbone True
```

Checkpointing & logging:
- Checkpoints are saved under exp/<experiment>/checkpoints
- TensorBoard logs under exp/<experiment>/runs
- We recommend saving best-by-IoU and periodic snapshots.

Evaluation metrics (per-pixel)
- Precision, Recall, F1-score (we use landslide class F1 as primary metric)
- Intersection over Union (IoU)
- Average Precision may be provided as supplementary

Inference & evaluation
----------------------
Inference scripts:
- Inference/predict.py
  - Accepts model checkpoint, test list, sliding-window settings, and outputs predicted masks (uint8 0/1).
- Inference/postprocess.py
  - Morphological filtering, small-object removal, vectorization helpers.

Example inference command:
```
python Inference/predict.py \
  --checkpoint ./exp/sugarfuse_run1/checkpoints/best_iou.pth \
  --data_root /path/to/GBMT-SLID \
  --test_list data/test_list.txt \
  --out_dir ./predictions/testset \
  --tile_size 256 --stride 128
```

Evaluation example:
```
python Inference/eval.py \
  --pred_dir ./predictions/testset \
  --gt_dir /path/to/GBMT-SLID/test/mask \
  --metrics f1,iou,precision,recall \
  --save_report ./exp/sugarfuse_run1/eval_report.json
```

Reproducibility & Unseen ROI inference
- We include scripts used in the paper to run inference on 5 unseen ROIs: DRC, Uganda, Myanmar, Philippines, Colombia.
- Use consistent preprocessing + cDMT translation (if used at inference) to harmonize inputs before prediction.

Results (summary from the manuscript)
- On GBMT‑SLID test set:
  - Recall: 94.6%
  - F1-score: 80.5%
  - IoU: 67.5%
- Generalization on five unseen regions:
  - Mean F1: 58.02%
  - Mean IoU: 41.08%
- Ablation highlights:
  - Topographic input significantly improves segmentation performance.
  - SGPA outperforms standard attention alternatives.
  - The full DES (cDMT + DDIM) improves F1 by 26.94% vs standard augmentation.

Files and folder structure
--------------------------
Top-level structure (what the repo will contain):
```
├── Data preprocessing/
├── Diffusion based data enhancement/
├── Models/
├── Training_validation/
├── Inference/
├── README.md
├── requirements.txt
├── create_repo_tree.sh
├── LICENSE
```
Each top directory contains scripts, notebooks, sample configs (YAML), and .gitkeep placeholders so code can be added incrementally.

Citation
--------
If you use this dataset and code, please cite the paper (add full bibtex after acceptance). Example (placeholder):
```
@article{your2025sugarfuse,
  author = {Nattabifir et al.},
  title = {SUGARFuseNet: Diffusion‑Driven Domain Adaptation and Bimodal Bitemporal Fusion for Advancing Global Landslide Segmentation on GBMT‑SLID},
  journal = {Under review / To appear},
  year = {2025}
}
```
Please replace with the final published citation when available.

License
-------
This repository is released under the MIT License (or choose appropriate license). See LICENSE file.

Acknowledgements
----------------
- Sentinel‑2 (ESA) and Copernicus DEM data sources
- Any funding and collaborators you want to acknowledge

Contributing
------------
Contributions are welcome: open issues for bugs/feature requests and submit PRs. Please follow the repository code style and include tests for new features where applicable.

Contact
-------
Author: Nattabifir  
GitHub: https://github.com/Nattabifir  
Email: (add your email if you wish)

