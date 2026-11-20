# CFAR ship detection on xView3-SAR: a two-scene benchmark

This folder measures how well the classic CFAR detector family used in this project finds vessels in Sentinel-1 VH imagery, using the real labels of the [xView3-SAR](https://iuu.xview.us/) validation set.

Three detectors are compared: **2D CA-CFAR**, **Hollow OS-CFAR** and **Two-Parameter CFAR**. Thresholds were chosen on one scene (calibration) and then run once, unchanged, on a second scene (test).

**Main result:** the highest-scoring detector here (2D CA-CFAR) reached F1 87.3% on the calibration scene but 69.2% on the test scene with the same locked threshold. The optimal threshold moved between scenes, so a fixed dB threshold does not transfer reliably.

## Files

| File | Content |
|---|---|
| `cfar_xview3_benchmark.ipynb` | Full pipeline and evaluation, saved with outputs |
| `results/pr_curves_590dd08f71056cacv.png` | Precision-recall curves, scene 1 (calibration) |
| `results/pr_curves_b1844cde847a3942v.png` | Precision-recall curves, scene 2 (test) |

## Protocol

- **Scenes:** `590dd08f71056cacv` (scene 1, calibration) and `b1844cde847a3942v` (scene 2, test), both from the xView3-SAR validation set.
- **Window:** 6000 × 6000 px (60 × 60 km) centred on the median position of labelled vessels, clipped to the image.
- **Preprocessing:** missing pixels set to -30 dB, 3 × 3 median filter on VH (dB). No absolute brightness cut-off (floor at -30 dB).
- **Water mask:** the dataset's `owiMask == 0` (about 500 m resolution), upsampled to the window. Labelled vessels on masked-out cells are reported as *not evaluated*, not as misses.
- **Detectors:**
  - 2D CA-CFAR: guard 9 × 9, background 25 × 25, locked T = 11.6 dB
  - Hollow OS-CFAR: guard 9 × 9, outer 21 × 21, 85th percentile, locked T = 8.1 dB
  - Two-Parameter CFAR: guard 9 × 9, background 25 × 25, locked T = 5.5 σ
  - Reference low threshold T = 5.0 dB (high-recall operating point) for CA and OS
- **Post-processing:** 5 × 5 binary closing, connected components, minimum bounding-box area 3 px, one centroid per object.
- **Matching:** one-to-one Hungarian matching, maximum distance 20 px (200 m).
- **Ignore rule:** predictions within 20 px of a labelled non-vessel object (`is_vessel` False or missing) are not counted as false alarms. In the HIGH-only tables, MEDIUM/LOW vessels are also ignored.
- **Intervals:** 95% Wilson intervals for precision and recall.
- The thresholds were fixed on scene 1 before scene 2 was run. The protocol note in the notebook was written afterwards.

## Coverage

| Scene | Labelled vessels in window | On masked-out cells (not evaluated) | Evaluated | of which HIGH confidence |
|---|---|---|---|---|
| Scene 1 (calibration) | 214 | 98 (46%) | 116 | 77 |
| Scene 2 (test) | 193 | 92 (48%) | 101 | 55 |

## Results

Scene 1 numbers are in-sample (the thresholds were chosen there). Scene 2 numbers are the out-of-sample test.

### Scene 2 (test, out-of-sample): all evaluated vessels (N = 101)

| Operating point | TP | FP | FN | Precision (95% CI) | Recall (95% CI) | F1 |
|---|---|---|---|---|---|---|
| 2D CA-CFAR, locked T = 11.6 | 72 | 35 | 29 | 67.3% (57.9-75.4) | 71.3% (61.8-79.2) | 69.2% |
| 2D CA-CFAR, ref T = 5.0 | 99 | 493 | 2 | 16.7% (13.9-19.9) | 98.0% (93.1-99.5) | 28.6% |
| Hollow OS-CFAR, locked T = 8.1 | 74 | 58 | 27 | 56.1% (47.5-64.2) | 73.3% (63.9-80.9) | 63.5% |
| Hollow OS-CFAR, ref T = 5.0 | 94 | 228 | 7 | 29.2% (24.5-34.4) | 93.1% (86.4-96.6) | 44.4% |
| Two-Param CFAR, locked T = 5.5 σ | 25 | 93 | 76 | 21.2% (14.8-29.4) | 24.8% (17.4-34.0) | 22.8% |

### Scene 2 (test): HIGH-confidence labels only (N = 55)

| Operating point | TP | FP | FN | Precision (95% CI) | Recall (95% CI) | F1 |
|---|---|---|---|---|---|---|
| 2D CA-CFAR, locked T = 11.6 | 51 | 33 | 4 | 60.7% (50.0-70.5) | 92.7% (82.7-97.1) | 73.4% |
| 2D CA-CFAR, ref T = 5.0 | 55 | 490 | 0 | 10.1% (7.8-12.9) | 100.0% (93.5-100.0) | 18.3% |
| Hollow OS-CFAR, locked T = 8.1 | 51 | 52 | 4 | 49.5% (40.1-59.0) | 92.7% (82.7-97.1) | 64.6% |
| Hollow OS-CFAR, ref T = 5.0 | 55 | 225 | 0 | 19.6% (15.4-24.7) | 100.0% (93.5-100.0) | 32.8% |
| Two-Param CFAR, locked T = 5.5 σ | 15 | 68 | 40 | 18.1% (11.3-27.7) | 27.3% (17.3-40.2) | 21.7% |

### Scene 1 (calibration, in-sample): all evaluated vessels (N = 116)

| Operating point | TP | FP | FN | Precision (95% CI) | Recall (95% CI) | F1 |
|---|---|---|---|---|---|---|
| 2D CA-CFAR, locked T = 11.6 | 96 | 8 | 20 | 92.3% (85.6-96.1) | 82.8% (74.9-88.6) | 87.3% |
| 2D CA-CFAR, ref T = 5.0 | 112 | 177 | 4 | 38.8% (33.3-44.5) | 96.6% (91.5-98.7) | 55.3% |
| Hollow OS-CFAR, locked T = 8.1 | 100 | 24 | 16 | 80.6% (72.8-86.6) | 86.2% (78.8-91.3) | 83.3% |
| Hollow OS-CFAR, ref T = 5.0 | 107 | 58 | 9 | 64.8% (57.3-71.7) | 92.2% (85.9-95.9) | 76.2% |
| Two-Param CFAR, locked T = 5.5 σ | 40 | 30 | 76 | 57.1% (45.5-68.1) | 34.5% (26.5-43.5) | 43.0% |

### Scene 1 (calibration): HIGH-confidence labels only (N = 77)

| Operating point | TP | FP | FN | Precision (95% CI) | Recall (95% CI) | F1 |
|---|---|---|---|---|---|---|
| 2D CA-CFAR, locked T = 11.6 | 71 | 9 | 6 | 88.8% (80.0-94.0) | 92.2% (84.0-96.4) | 90.4% |
| 2D CA-CFAR, ref T = 5.0 | 77 | 173 | 0 | 30.8% (25.4-36.8) | 100.0% (95.2-100.0) | 47.1% |
| Hollow OS-CFAR, locked T = 8.1 | 72 | 23 | 5 | 75.8% (66.3-83.3) | 93.5% (85.7-97.2) | 83.7% |
| Hollow OS-CFAR, ref T = 5.0 | 76 | 56 | 1 | 57.6% (49.0-65.7) | 98.7% (93.0-99.8) | 72.7% |
| Two-Param CFAR, locked T = 5.5 σ | 19 | 30 | 58 | 38.8% (26.4-52.8) | 24.7% (16.4-35.4) | 30.2% |

### Threshold drift (all evaluated vessels)

The best threshold is found by sweeping on the same scene it is reported for, so it is an optimistic upper bound.

| Detector | Locked T (from scene 1) | Scene 2 F1 at locked T | Best T on scene 2 | Scene 2 best F1 (optimistic) | Loss from drift |
|---|---|---|---|---|---|
| 2D CA-CFAR | 11.6 dB | 69.2% | 13.3 dB | 76.7% | -7.4 points |
| Hollow OS-CFAR | 8.1 dB | 63.5% | 11.6 dB | 74.2% | -10.7 points |
| Two-Param CFAR | 5.5 σ | 22.8% | 9.0 σ | 39.7% | -16.9 points |

On scene 1 the best thresholds were the locked ones (CA 11.6 dB, OS 8.1 dB, Two-Param 5.5 σ). Note that 9.0 σ is the upper end of the Two-Param sweep grid, so its true optimum on scene 2 may be higher.

## Findings

1. **Fixed thresholds do not transfer reliably.** All three detectors needed a higher threshold on scene 2. Holding the scene 1 threshold cost 7 to 17 F1 points.
2. **2D CA-CFAR had the highest F1 at the locked thresholds on both scenes and the smallest loss from drift**, but its lead over OS-CFAR is within the confidence intervals. Two-Parameter CFAR was far behind on both.
3. **HIGH-confidence vessels are found well on both scenes** (recall about 92-93% at the locked thresholds). Most misses on scene 2 are MEDIUM/LOW-confidence labels. This is partly expected: HIGH labels come from agreement between an automated pipeline (which itself uses CFAR-style detection) and human review.
4. **The low reference threshold (T = 5.0 dB)** finds almost every evaluated vessel (CA: 98% recall on scene 2) at the cost of many false alarms (493 on a 60 × 60 km window). It could serve as a first-stage candidate generator for a second-stage classifier.
5. **About half of the labelled vessels in each window were not evaluated** because the coarse water mask marks their cells as land. Near-shore and harbour areas are outside the scope of these results.

## Limitations

- Two scenes and about 100 evaluated vessels each: confidence intervals are wide, and differences of a few F1 points are within noise.
- Unlabelled real objects are counted as false alarms, so precision is likely underestimated. The ignore rule is not one-to-one, which can slightly favour the detectors.
- Results say nothing about vessels on masked-out cells (see Coverage).
- This benchmark measures detection only. It does not evaluate any AIS-based "dark vessel" logic, and the label column `source` should not be read as "AIS on/off".
- Numbers are not comparable with the official xView3 leaderboard (different subset, own matching rule, official scoring code not used).

## How to reproduce

1. Register on the official xView3 site and download, under its terms, the validation labels (`validation.csv`) and a download-link file (`tiny.txt`) that contains links for the two scene ids above.
2. Open `cfar_xview3_benchmark.ipynb` (for example in Google Colab) and place `validation.csv` and `tiny.txt` next to it.
3. Run all cells. The download cell fetches and extracts the two scenes (scene 1 is about 1.3 GB compressed) without printing the signed links.

Do not commit the data, `tiny.txt` or any signed download link.

## Data credit

xView3-SAR: Paolo et al., "xView3-SAR: Detecting Dark Fishing Activity Using Synthetic Aperture Radar Imagery", NeurIPS 2022 Datasets and Benchmarks Track.
