# 🛰️ Sentinel-1 SAR: Autonomous "Dark Vessel" Detection Pipeline

![Python](https://img.shields.io/badge/Python-3.10+-blue.svg)
![SAR](https://img.shields.io/badge/Data-Sentinel--1%20SAR-lightgrey.svg)
![Algorithm](https://img.shields.io/badge/Algorithm-2D%20CA--CFAR-red.svg)
![Status](https://img.shields.io/badge/Status-v2.0%20Land%20Masking%20Validated-success.svg)

## 📌 Executive Summary
Optical satellite imagery (like Sentinel-2) suffers from "Spatial Blindness" due to cloud cover and night-time limitations. This project overcomes these physical barriers by utilizing **Synthetic Aperture Radar (SAR)** data from Sentinel-1 to autonomously detect metallic structures (ships) in open oceans, regardless of weather or daylight.

Going beyond simple detection, this pipeline features a simulated **Decision Support System (DSS)** that cross-references radar targets with AIS (Automatic Identification System) databases to identify **"Dark Vessels"**—ships that have intentionally disabled their tracking radios, often associated with illegal fishing, smuggling, or embargo evasion.

## ⚙️ System Architecture (End-to-End Pipeline)

1. **Autonomous Data Ingestion:** Connects directly to the Microsoft Planetary Computer API, fetches Sentinel-1 RTC (Radiometrically Terrain Corrected) data, resolves CRS/Projection conflicts dynamically, and converts raw linear backscatter into logarithmic Decibel (dB) scale.
2. **Signal Processing:** Applies morphological Median Speckle Filtering to smooth out sea-surface noise and wave clutters while preserving high-intensity metallic backscatters (Volume Scattering in the VH polarization).
3. **Tactical Detection (CA-CFAR):** Implements a military-standard **2D Cell-Averaging Constant False Alarm Rate (CA-CFAR)** algorithm. Instead of static thresholds, it dynamically calculates local ocean background noise to maintain a stable detection rate across varying weather conditions.
4. **Target Clustering & Bounding:** Connects activated pixels, filters out single-pixel anomalies (sea foam), and generates bounding boxes around valid metallic signatures.
5. **Land Masking (v2.0):** Converts each detection's pixel coordinates to real-world geographic coordinates (via affine transform scaling), then cross-references them against Natural Earth coastline data (GeoPandas/Shapely) to programmatically drop any detections falling on land — *before* they reach the AIS cross-check stage.
6. **Intelligence Cross-Checking (AIS Mock):** Extracts the spatial coordinates of the remaining, confirmed-maritime targets and simulates an API call to a global AIS database to classify each as either a `🟢 Legal Vessel` or a `🔴 Dark Vessel`.

## 📊 Tactical Dashboard Preview

| 1. Filtered SAR Signal | 2. Target Clustering | 3. Tactical Intelligence |
| :---: | :---: | :---: |
| ![Filtered SAR Signal](https://github.com/user-attachments/assets/35258502-6944-47fb-a027-e93ad0ddd689) | ![CA-CFAR Detections](https://github.com/user-attachments/assets/c4675c32-aa17-47c0-b818-73a43082f523) | ![Dark Vessel Isolation](https://github.com/user-attachments/assets/c5b43f56-28a3-457f-a474-50608c1ae671) |
| *Smoothed VH Polarization* | *CA-CFAR Detections* | *Dark Vessel Isolation* |

## 🚀 How to Run (Proof of Concept)
The entire pipeline is compiled into a single Jupyter Notebook for seamless execution.
1. Clone the repository.
2. Install the required spatial dependencies: `pip install pystac-client planetary-computer rasterio scipy matplotlib numpy geopandas shapely`
3. Run the notebook sequentially. The system is currently locked onto the **Istanbul Ahirkapi Anchorage Area** (`bbox: [28.85, 40.85, 29.05, 40.95]`) to demonstrate high-density traffic detection.

---

## ✅ v2.0 Update — Land Masking Implemented & Validated

The land false-alarm issue described in v1.0 (below) has been addressed with a **Static Land Masking Pipeline**: each CFAR detection's pixel coordinates are converted to real-world lat/lon (accounting for the CRS of the native SAR projection and the resampling scale factor between the read window and the output array), then checked against Natural Earth 10m coastline polygons using a point-in-polygon test.

**Validation results, tested on two distinct regions:**

| Region | Raw Detections | Land-Based (Removed) | Confirmed Maritime |
|---|---|---|---|
| Ahırkapı Anchorage (open water) | 109 | 0 | 109 |
| Bosphorus shoreline (mixed land/sea) | 3,826 | 3,263 | 563 |

The open-water test confirms the pipeline correctly leaves valid maritime detections untouched. The coastal test confirms the land-masking logic actively identifies and removes land-based false alarms (coastal structures, buildings) that the CA-CFAR algorithm — by design — cannot distinguish from vessels on its own.

### Known limitation (v2.0)
Land-masking accuracy is bounded by the resolution of the Natural Earth 10m coastline dataset. Small man-made structures (piers, breakwaters, docked vessels directly adjacent to shore) may not be captured with pixel-level precision. A higher-resolution coastline source (e.g. OpenStreetMap water polygons) is a candidate for v3.0.

---

## 🚧 v1.0 Original Limitation (Resolved in v2.0)

The initial PoC successfully detected and cross-referenced targets but had a known architectural vulnerability: the CA-CFAR algorithm is highly sensitive to dense metallic/concrete structures, so target boxes falling on a coastline, port crane, or urban area were misclassified as "Dark Vessels" (since static buildings emit no AIS signal). This is the issue resolved above.
