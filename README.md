# 🛰️ Sentinel-1 SAR: Autonomous "Dark Vessel" Detection Pipeline

![Python](https://img.shields.io/badge/Python-3.10+-blue.svg)
![SAR](https://img.shields.io/badge/Data-Sentinel--1%20SAR-lightgrey.svg)
![Algorithm](https://img.shields.io/badge/Algorithm-2D%20CA--CFAR-red.svg)
![Status](https://img.shields.io/badge/Status-PoC%20Completed-success.svg)

## 📌 Executive Summary
Optical satellite imagery (like Sentinel-2) suffers from "Spatial Blindness" due to cloud cover and night-time limitations. This project overcomes these physical barriers by utilizing **Synthetic Aperture Radar (SAR)** data from Sentinel-1 to autonomously detect metallic structures (ships) in open oceans, regardless of weather or daylight. 

Going beyond simple detection, this pipeline features a simulated **Decision Support System (DSS)** that cross-references radar targets with AIS (Automatic Identification System) databases to identify **"Dark Vessels"**—ships that have intentionally disabled their tracking radios, often associated with illegal fishing, smuggling, or embargo evasion.

## ⚙️ System Architecture (End-to-End Pipeline)

1. **Autonomous Data Ingestion:** Connects directly to the Microsoft Planetary Computer API, fetches Sentinel-1 RTC (Radiometrically Terrain Corrected) data, resolves CRS/Projection conflicts dynamically, and converts raw linear backscatter into logarithmic Decibel (dB) scale.
2. **Signal Processing:** Applies morphological Median Speckle Filtering to smooth out sea-surface noise and wave clutters while preserving high-intensity metallic backscatters (Volume Scattering in the VH polarization).
3. **Tactical Detection (CA-CFAR):** Implements a military-standard **2D Cell-Averaging Constant False Alarm Rate (CA-CFAR)** algorithm. Instead of static thresholds, it dynamically calculates local ocean background noise to maintain a stable detection rate across varying weather conditions.
4. **Target Clustering & Bounding:** Connects activated pixels, filters out single-pixel anomalies (sea foam), and generates bounding boxes around valid metallic signatures.
5. **Intelligence Cross-Checking (AIS Mock):** Extracts the spatial coordinates of detected targets and simulates an API call to a global AIS database to classify the target as either a `🟢 Legal Vessel` or a `🔴 Dark Vessel`.

## 📊 Tactical Dashboard Preview

| 1. Filtered SAR Signal | 2. Target Clustering | 3. Tactical Intelligence |
| :---: | :---: | :---: |
| ![Filtered SAR Signal](https://github.com/user-attachments/assets/35258502-6944-47fb-a027-e93ad0ddd689) | ![CA-CFAR Detections](https://github.com/user-attachments/assets/c4675c32-aa17-47c0-b818-73a43082f523) | ![Dark Vessel Isolation](https://github.com/user-attachments/assets/c5b43f56-28a3-457f-a474-50608c1ae671) |
| *Smoothed VH Polarization* | *CA-CFAR Detections* | *Dark Vessel Isolation* |

## 🚀 How to Run (Proof of Concept)
The entire pipeline is compiled into a single Jupyter Notebook for seamless execution.
1. Clone the repository.
2. Install the required spatial dependencies: `pip install pystac-client planetary-computer rasterio scipy matplotlib numpy`
3. Run the notebook sequentially. The system is currently locked onto the **Istanbul Ahirkapi Anchorage Area** (`bbox: [28.85, 40.85, 29.05, 40.95]`) to demonstrate high-density traffic detection.

---

## 🚧 Known Limitations & Future Roadmap (v2.0)

While the current PoC successfully detects and cross-references targets, it possesses a known architectural vulnerability regarding radar physics:

* **Land False Alarms:** The CA-CFAR algorithm is highly sensitive to dense metallic/concrete structures. Currently, if the target bounding box falls on a coastline, port crane, or urban area, the system will misclassify it as a "Dark Vessel" because static buildings do not emit AIS signals. 
* **Proposed Solution (v2.0):** Integration of a **Static Land Masking Pipeline**. In the next iteration, I plan to integrate `GeoPandas` and global coastline vector data (Shapefiles/GeoJSON). By performing a spatial intersection, any radar coordinates falling within land boundaries will be programmatically dropped *before* the CFAR and AIS cross-referencing phases, effectively dropping the land false-alarm rate to 0%.
