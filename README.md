# GulfMetric — The Road as a Ruler

**Training-free ground-plane scale correction for a monocular 3D foundation model.**
CSBP441 Applied Computer Vision, Fall 2026, UAE University.

A frozen [MoGe-2](https://github.com/microsoft/MoGe) model predicts a metric 3D point map from one image, but its absolute scale is often wrong. When the camera height above the road is known (1.65 m on KITTI), we fit the road plane in the model's own prediction with RANSAC, read how high the model thinks the camera is (`h_pred`), and rescale everything by `s = 1.65 / h_pred`. This is **Geometric Scale Correction (GSC)**. There is no training; the step runs as post-processing.

## Results (900 KITTI test images)

| Variant | AbsRel ↓ | RMSE (m) ↓ | δ<1.25 (%) ↑ | vs. (a) |
|---|---|---|---|---|
| (a) MoGe-2 baseline | 0.222 | 4.62 | 39.6 | — |
| (b) + true focal length | 0.133 | 3.86 | 86.0 | −40% |
| **(c) + GSC** | **0.119** | **3.24** | **92.4** | **−46%** |
| (d) + true focal + GSC | 0.148 | 4.24 | 82.1 | −33% |
| (d') (d) with least-squares plane, no RANSAC | 0.413 | 7.70 | 51.0 | +86% |
| *(c) + plausibility check (0.5 ≤ s ≤ 2), added after failure analysis* | *0.097* | *2.97* | *92.9* | *−57%* |
| Oracle median scaling (uses ground truth, upper bound) | 0.055 | 2.58 | 97.4 | −75% |

- **Consistency:** GSC (c) improves **97.4%** of test images.
- **Cost:** GSC adds **36 ms/image** (median 31 ms) on a Tesla T4, against ~380–440 ms for MoGe-2 itself; peak GPU memory is 2.38 GB.
- **Main failure:** streets lined with parked cars (`drive_0036`), where RANSAC fits the cars instead of the road.

Full analysis: [`report/`](report/).

## Repository structure

```
notebooks/GulfMetric_pipeline.ipynb   full pipeline: setup, MoGe-2, GSC, evaluation, figures, demo (with outputs)
results/                              CSV result tables produced by the notebook
  main_table.csv                      Table 1: all variants
  per_image_test.csv                  every metric for all 900 test images
  dev_threshold_sweep.csv             RANSAC threshold selection on the dev split
  cost_table.csv, by_distance.csv     runtime / memory, error by distance
  failure_cases.csv                   worst cases
  sanity_check_table.csv              plausibility-check rows
  figures/                            bar chart, qualitative and failure figures
splits/dev.txt, splits/test.txt       fixed split (100 dev / 900 test image names)
report/                               LaTeX source of the report (CVPR template)
```

## How to reproduce

1. **Open the notebook in Google Colab** and set Runtime → Change runtime type → **T4 GPU**.
2. **Get the data:** the KITTI depth "selected validation" set, about 2 GB.
   ```bash
   wget https://s3.eu-central-1.amazonaws.com/avg-kitti/data_depth_selection.zip
   unzip -q data_depth_selection.zip "depth_selection/val_selection_cropped/*"
   ```
   If the download fails, register at the [KITTI depth benchmark](https://www.cvlibs.net/datasets/kitti/eval_depth.php?benchmark=depth_prediction). Put `val_selection_cropped/` at `GulfMetric/kitti/val_selection_cropped/` in Google Drive, and the files from `splits/` at `GulfMetric/splits/`.
3. **Run the cells in order.**
   - Cells 1–5: setup and functions, ~5 min including the MoGe-2 install and weight download.
   - Cell 6: chooses the RANSAC threshold on the 100 dev images, ~5 min.
   - Cell 7: main run on the 900 test images, ~51 min on a T4. **Resume-safe:** after a disconnect, rerun cells 1–5 and 7.
   - Cells 8–12: tables, plausibility check, figures, failure cases, demo.
4. Outputs are written to `GulfMetric/results/` in Google Drive.

**Settings:**
- checkpoint `Ruicheng/moge-2-vitl` (fp16)
- camera height 1.65 m
- ground candidates = bottom third of the image
- RANSAC: 300 hypotheses, ≤ 20,000 points, |n_y| > 0.8, threshold **0.2 m** (chosen on dev), ≥ 500 inliers
- metrics on LiDAR pixels in (0, 80] m

**Software:** Python 3, PyTorch, NumPy, OpenCV, pandas, Matplotlib, Pillow, MoGe (`pip install git+https://github.com/microsoft/MoGe.git`). See `requirements.txt`.

## Attribution

- **Built in this project:** the GSC step (ground selection, constrained RANSAC, rescaling, fallback), the plausibility check, the evaluation pipeline (split, metrics, ablation, timing) and all experiments.
- **External:**
  - MoGe-2 code and weights (Microsoft; Wang et al., 2025)
  - the KITTI depth benchmark (Geiger et al., CVPR 2012; Uhrig et al., 3DV 2017; CC BY-NC-SA 3.0, not redistributed here)
  - PyTorch, NumPy, OpenCV, pandas, Matplotlib, Google Colab
- **Related method:** ground-plane scale recovery follows Song & Chandraker (CVPR 2014); see also GroCo (ECCV 2024).
- **AI assistance:** an AI assistant (Claude, Anthropic) helped write code and draft documentation; all experiments were run and checked by the team.

## Team

Makhmud · Mohamed Ibrahim Almehrzi · Abdalkarim Abumashayekh · Mohammed Hussein
