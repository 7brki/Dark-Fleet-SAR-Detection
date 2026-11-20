# Dark-Fleet-SAR-Detection: CFAR ship detection on Sentinel-1 SAR

![Python](https://img.shields.io/badge/Python-3.10+-blue.svg)
![Data](https://img.shields.io/badge/Data-Sentinel--1%20SAR-lightgrey.svg)
![Status](https://img.shields.io/badge/Status-research%20prototype-orange.svg)

A small, end-to-end ship detection pipeline for Sentinel-1 radar imagery, built as a learning project around the "dark vessel" problem: ships that are visible on radar but do not broadcast their position over AIS.

The repository has two parts:

| Part | What it is | What is real, what is not |
|---|---|---|
| [`Dark_Fleet_SAR_Detection_v2.ipynb`](Dark_Fleet_SAR_Detection_v2.ipynb) | Case study on one Sentinel-1 scene over the Istanbul Ahırkapı anchorage: download, speckle filtering, CFAR detection, land masking, and a placeholder AIS step | Detection and land masking run on real data. **The AIS step is a random placeholder**, so the "legal / dark" labels it produces carry no information. |
| [`benchmark/`](benchmark/) | Evaluation of the same CFAR detector family against real labels from the xView3-SAR dataset, with thresholds chosen on one scene and tested on another | Real data, real labels, locked thresholds. |

## Key result: how well does the detector work?

The Istanbul notebook has no ground truth, so its detections cannot be scored. To measure the detector, the same family of CFAR detectors was evaluated on two scenes of the [xView3-SAR](https://iuu.xview.us/) validation set. Thresholds were chosen on scene 1 and then run unchanged on scene 2.

| Detector | Scene 1 F1 (in-sample) | Scene 2 F1 (locked threshold) | Scene 2 F1 (best threshold, optimistic) |
|---|---|---|---|
| 2D CA-CFAR | 87.3% | 69.2% | 76.7% |
| Hollow OS-CFAR | 83.3% | 63.5% | 74.2% |

Performance dropped on the unseen scene and the best threshold moved between scenes, so a fixed dB threshold does not transfer reliably. This also means the fixed threshold used in the Istanbul notebook is not validated. Full tables, confidence intervals, protocol and limitations are in [`benchmark/README.md`](benchmark/README.md).

Note that the benchmark detectors use their own window sizes and thresholds (for example CA-CFAR with a 9 × 9 guard window, a 25 × 25 background window and T = 11.6 dB), which differ from the detector in the Istanbul notebook described below (5 × 5 guard window, 15 × 15 background window, 4 dB).

## Istanbul case study: pipeline

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/7brki/Dark-Fleet-SAR-Detection/blob/main/Dark_Fleet_SAR_Detection_v2.ipynb)

- **Area and data:** bbox `[28.85, 40.85, 29.05, 40.95]` (Ahırkapı anchorage, Sea of Marmara), date range 2023-08-01 to 2023-08-15. The first matching Sentinel-1 RTC scene from Microsoft Planetary Computer is used (`S1A_IW_GRDH_1SDV_20230811T160002_...`).

1. **Read and convert:** VV and VH are read for the bbox window, resampled to 1000 × 1000 pixels, and converted to dB.
2. **Speckle filtering:** 3 × 3 median filter on VH. VH (cross-polarisation) is used because sea clutter is usually weaker there, so ships stand out more.
3. **Detection (CA-CFAR with guard cells, v2.1):** the background of each pixel is the mean of a ring: a 15 × 15 window minus the 5 × 5 guard window around the pixel under test (200 pixels). Pixels more than 4 dB above that ring mean are marked as detections.
4. **Clustering:** connected detection pixels are grouped, and groups with a bounding box smaller than 2 pixels are dropped.
5. **Land masking (v2.0):** the centre of each bounding box is converted from pixel coordinates to longitude/latitude (accounting for the scene CRS and the resampling factor) and tested against Natural Earth 10 m land polygons. Detections on land are removed.
6. **AIS placeholder:** each remaining detection is labelled "legal" or "dark" at random (80% / 20%). No AIS data is used.

## Istanbul results

On the default run the notebook finds **119 detections** in the anchorage area (v2.0 found 109). These are candidates, not confirmed vessels: there is no ground truth for this scene.

Detection and land masking were checked on two areas, with the v2.0 detector (no guard cells) and the v2.1 detector (guard cells):

| Area | Detector | Raw detections | Removed as land | Remaining |
|---|---|---|---|---|
| Ahırkapı anchorage (open water, default notebook run) | v2.0 | 109 | 0 | 109 |
| Ahırkapı anchorage (open water, default notebook run) | v2.1 | 119 | 0 | 119 |
| Bosphorus shoreline, bbox `[28.90, 40.95, 29.10, 41.05]` (same notebook, only the bbox changed) | v2.0 | 3,826 | 3,263 | 563 |
| Bosphorus shoreline, bbox `[28.90, 40.95, 29.10, 41.05]` (same notebook, only the bbox changed) | v2.1 | 3,990 | 3,415 | 575 |

All runs use the same Sentinel-1 scene (`S1A_IW_GRDH_1SDV_20230811T160002_...`) and were run in November 2026; the v2.0 shoreline numbers match an earlier run exactly.

With guard cells, a bright target no longer raises its own background, so more pixels pass the threshold and the detector finds more candidates (+10 in the anchorage). Without ground truth it is not known whether the extra candidates are vessels.

The shoreline run shows that the land mask removes many land-based false alarms (buildings, port structures). It does not show that the remaining 563 detections are ships.

Figures below are from the v2.0 run; the notebook itself contains the v2.1 figures.

| Median-filtered VH | CFAR detections | Random "AIS" labels (placeholder) |
| :---: | :---: | :---: |
| ![Filtered SAR Signal](https://github.com/user-attachments/assets/35258502-6944-47fb-a027-e93ad0ddd689) | ![CA-CFAR Detections](https://github.com/user-attachments/assets/c4675c32-aa17-47c0-b818-73a43082f523) | ![Random AIS labels](https://github.com/user-attachments/assets/c5b43f56-28a3-457f-a474-50608c1ae671) |

## Known limitations (Istanbul notebook)

- **Resampling distorts the grid.** The window is about 17 km × 11 km but is resampled to 1000 × 1000 pixels, so pixels are roughly 17 m × 11 m instead of the native 10 m × 10 m.
- **Coarse land mask, applied late.** Natural Earth 10 m polygons miss small structures such as piers, breakwaters and small islands, and only the box centre is tested. The mask is applied after detection rather than before it. The land polygons are also clipped to a fixed box (`[28.85, 40.90, 29.15, 41.10]`), so other areas need that box changed.
- **No ground truth.** Detections in Istanbul are not scored; the xView3 benchmark is the only quantitative evaluation in this repository.
- **The AIS step is a random placeholder** (fixed seed, so the output is repeatable). Its "AIS match / no AIS" labels carry no information.

## Roadmap

1. Evaluate on more xView3 scenes (two scenes are not enough for general conclusions).
2. Use a higher-resolution coastline (for example OpenStreetMap water polygons) and apply the mask before detection.
3. Use a low CFAR threshold as a candidate generator and a small CNN to classify the candidates, trained and tested on separate scenes.
4. Replace the AIS placeholder with real AIS data.

## History

- **v1.0:** first pipeline. Land structures (coastline, cranes, buildings) were detected as targets and labelled as "dark vessels".
- **v2.0:** land masking with Natural Earth polygons.
- **v2.1 (November 2026):** guard cells added to the CA-CFAR detector (109 → 119 detections in the anchorage).
- **Benchmark (November 2026):** two-scene evaluation on xView3-SAR, see [`benchmark/`](benchmark/).

## Data and credits

- Sentinel-1 RTC data: Copernicus Sentinel data, accessed through Microsoft Planetary Computer.
- Land polygons: Natural Earth 10 m.
- Benchmark labels and scenes: xView3-SAR (Paolo et al., NeurIPS 2022 Datasets and Benchmarks Track).
