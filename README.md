# VUDO - Vegetazione Urbana Da Ortofoto

**Vegetation mapping from AGEA orthophotos: a reproducible geospatial pipeline for Italian municipalities**

---

## Table of Contents

- [1. Introduction](#1-introduction)
- [2. Retrieving Input Orthophotos](#2-retrieving-input-orthophotos)
  - [2.1 AGEA GeoPortale (national, open data)](#21-agea-geoportale-national-open-data)
  - [2.2 Historical or restricted orthophotos](#22-historical-or-restricted-orthophotos)
  - [2.3 Preparing the input data](#23-preparing-the-input-data)
- [3. Repository Structure](#3-repository-structure)
- [4. Script Descriptions](#4-script-descriptions)
  - [VUDO-1-Create_NDVI.ipynb](#vudo-1-create_ndviipynb)
  - [VUDO-2-First_Threshold.ipynb](#vudo-2-first_thresholdipynb)
  - [VUDO-3-Refined_Threshold.ipynb](#vudo-3-refined_thresholdipynb)
  - [VUDO-4-Clumping_Mask.ipynb](#vudo-4-clumping_maskipynb)
  - [VUDO-5-Clumping_Stats.ipynb](#vudo-5-clumping_statsipynb)
- [5. Usage Notes and Caveats](#5-usage-notes-and-caveats)
- [6. License and Attribution](#6-license-and-attribution)
- [7. References](#7-references)

---

## 1. Introduction

**VUDO** (Vegetazione Urbana Da Ortofoto) is a geospatial data-processing pipeline that automatically maps urban vegetation from high-resolution aerial orthophotos. It takes RGBI (Red-Green-Blue-Infrared) orthophotos from AGEA—the Italian Agency for Agricultural Payments—and produces binary vegetation masks at the municipal scale, along with summary statistics on green cover and vegetation patch sizes.

The project is designed to be **city-agnostic** and **reproducible**: a single configuration block per notebook (municipality ISTAT code, index type, flight year) drives the entire workflow, and each step reads the output of the previous one. It is intended for urban planners, environmental researchers, and GIS analysts who need a transparent, code-driven alternative to manual vegetation digitization.

The pipeline implements several classification strategies:

- **KDE/KMeans thresholding** (Step 2): unsupervised clustering of index values to find vegetation/non-vegetation thresholds.
- **Australian adaptive pipeline** (Step 3): Canny edge detection + Otsu thresholding, following the method described in [this MDPI paper](https://www.mdpi.com/2072-4292/8/5/386).
- **Clump-size filtering** (Step 4): removal of vegetation patches smaller than a minimum area (default ~100 m²) via connected-component labeling.
- **Clump statistics** (Step 5): size-distribution analysis of the remaining vegetation patches.

All intermediate and final outputs are GeoTIFFs or CSVs written to `output/<procom>/`, where `<procom>` is the ISTAT municipality code.

---

## 2. Retrieving Input Orthophotos

The pipeline expects **RGBI (4-band) orthophotos** covering a single municipality. These are not bundled with the repository and must be downloaded separately.

### 2.1 AGEA GeoPortale (national, open data)

AGEA has activated a public GeoPortale for consultation and download of national orthophotos. As of the current release, **2022, 2023, and 2024** orthophotos are available, at **20 cm ground resolution**.

**Download procedure**:

1. Go to the **AGEA GeoPortale** (search for "GeoPortale AGEA" or access via the AGEA website).
2. Select the **flight year** (2022, 2023, or 2024). This determines which regions are available.
3. Navigate the map to your municipality, or use the address/locality search.
4. Select the **tiles** covering your area of interest. You can:
   - Click tiles directly on the map.
   - Draw a polygon/rectangle/circle to select an area.
   - **Maximum 20 tiles per request.**
5. Enter your **email address**, accept the terms, and complete the hCaptcha.
6. You will receive an **OTP (One-Time Password)** by email to verify your address.
7. After verification, a **download link** is sent to your email.

**License**: As of September 2026, AGEA orthophotos are released under **CC BY 4.0**, meaning they can be freely reused (including commercially) with attribution to AGEA.

**Coordinate system**: The GeoPortale does not explicitly declare the CRS. Based on user reports, orthophotos work correctly with **UTM 32N WGS84 (EPSG:32632)** for Tuscany and central Italy. You should verify the CRS of your downloaded file in QGIS or GDAL before using it in this pipeline.

### 2.2 Historical or restricted orthophotos

For older years (2015, 2018, 2021) or if you need the **IR (infrared) band** (which the standard GeoPortale RGB downloads may not include), orthophotos are typically distributed through regional geoportals or AGEA directly:

- **Regional WMS services**: Many regions (Emilia-Romagna, Piemonte, Toscana, etc.) expose AGEA orthophotos as WMS layers. However, WMS is for visualization only—you cannot download the raw raster.
- **Direct request to AGEA**: For public administration, CTU/CTP, or judicial purposes, you can request orthophotos via PEC (`protocollo@pec.agea.gov.it`) using the form on the AGEA website. For private/commercial use, AGEA does not distribute directly due to licensing restrictions on older flights.

**Critical for this pipeline**: VUDO **requires the IR (Near-Infrared) band** to compute NDVI. The standard AGEA GeoPortale RGB download may only have 3 bands. If your orthophoto lacks an IR band, you cannot compute NDVI with the formulas in Step 1. Check your file's band count (e.g., `gdalinfo` or QGIS layer properties) before proceeding.

### 2.3 Preparing the input data

Once you have your orthophotos, organize them as follows:

```
input/<procom>/
├── RGBI/          # One or more 4-band GeoTIFFs (R, G, B, IR)
└── SHP/           # Exactly ONE shapefile (.shp) of the municipality boundary
```

**Important**:

- The RGBI folder can contain **multiple tiles**; the pipeline mosaics them automatically.
- The SHP folder must contain **exactly one** `.shp` file, or the notebook will raise an assertion error.
- The municipality boundary can be downloaded from ISTAT or from your regional geoportal.

---

## 3. Repository Structure

```
JOS_AREEVERDI/
├── input/
│   └── <procom>/
│       ├── RGBI/          # Input RGBI orthophotos (user-provided)
│       └── SHP/           # Municipality boundary shapefile (user-provided)
├── output/
│   └── <procom>/          # All outputs for this municipality
│       ├── <city>-<procom>-<index>-<year>.tif                       # Index mosaic (Step 1)
│       ├── <city>-<procom>-<index>-<year>--KDE_mask.tif             # KDE mask (Step 2)
│       ├── <city>-<procom>-<index>-<year>--KMeans_mask.tif          # KMeans mask (Step 2)
│       ├── <city>-<procom>-<index>-<year>--Australian_mask.tif      # Australian mask (Step 3)
│       ├── <city>-<procom>-<index>-<year>-<method>_clump.tif        # Final clumped mask (Step 4)
│       ├── <city>-<procom>-<index>-<year>.csv                       # Summary stats (Steps 2–4)
│       └── <city>-<procom>-<index>-<year>-<method>--clump_stats.csv # Clump stats (Step 5)
├── VUDO-1-Create_NDVI.ipynb
├── VUDO-2-First_Threshold.ipynb
├── VUDO-3-Refined_Threshold.ipynb
├── VUDO-4-Clumping_Mask.ipynb
└── VUDO-5-Clumping_Stats.ipynb
```

All notebooks share a common `base_path` (e.g., `C:/Users/UTENTE/Downloads/JOS_areeverdi/`) that must be set consistently across all five files.

---

## 4. Script Descriptions

### VUDO-1-Create_NDVI.ipynb

**Purpose**: Crop orthophotos to the municipal boundary, compute a vegetation index (NDVI, ENDVI, etc.) per tile, and mosaic everything into a single city-wide GeoTIFF.

**Input**:

- `input/<procom>/RGBI/*.tif` — one or more 4-band RGBI GeoTIFFs.
- `input/<procom>/SHP/*.shp` — exactly one municipality boundary.

**Output**:

- `output/<procom>/<city>-<procom>-<index>-<year>.tif` — single-band Float32 vegetation index mosaic.
- (Intermediate files are automatically deleted after the mosaic is written.)

**Key operations**:

1. Reads the first RGBI tile to determine the source CRS (e.g., RDN2008 / UTM 33N), then looks it up in a `CRScorrespondences` dictionary.
2. Reprojects the municipality boundary to match the raster CRS.
3. Crops each RGBI tile to the boundary.
4. Computes the selected index per tile using `evaluate_index()`.
5. Merges all per-tile index rasters into a city-wide mosaic.
6. Deletes intermediate cropped tiles and per-tile rasters to leave only the final mosaic.

---

### VUDO-2-First_Threshold.ipynb

**Purpose**: Produce two independent binary vegetation masks via **KDE thresholding** and **KMeans clustering**, and write summary statistics.

**Input**:

- `output/<procom>/<city>-<procom>-<index>-<year>.tif` — the index mosaic from Step 1.

**Output**:

- `<city>-<procom>-<index>-<year>--KDE_mask.tif` — binary mask (0 = non-veg, 1 = veg, 255 = NoData).
- `<city>-<procom>-<index>-<year>--KMeans_mask.tif` — binary mask.
- `<city>-<procom>-<index>-<year>.csv` — summary stats (threshold, green pixel count/%, area in m² and ha, elapsed time).

**Key operations**:

1. Samples 1% of valid pixels (excluding NoData `-20`) to reduce memory usage.
2. Fits a Kernel Density Estimate (KDE) on the sampled values; finds local minima and maxima.
3. Filters out spurious extrema (those below 1/40 of the maximum density).
4. Uses the filtered maxima as KMeans cluster centroids (or falls back to a heuristic if only one maximum is found).
5. Applies both thresholds to the full-resolution raster, producing two masks.
6. Exports masks as compressed Byte GeoTIFFs and a CSV summary.

---

### VUDO-3-Refined_Threshold.ipynb

**Purpose**: Implement the **adaptive Australian pipeline** (Canny edge detection + Otsu thresholding) to produce a refined vegetation mask. This method is more selective than KDE/KMeans and tends to exclude spurious green pixels.

**Input**:

- The index mosaic from Step 1.
- The CSV from Step 2 (to read the KDE/KMeans threshold for edge filtering).

**Output**:

- `<city>-<procom>-<index>-<year>--Australian_mask.tif` — binary mask.
- Appends a row to the existing CSV with the Australian method's stats.

**Key operations**:

1. Loads the threshold from Step 2's CSV (`threshold_method = 'auto'` uses KDE, falling back to KMeans if the KDE threshold ≤ 0).
2. Runs **Canny edge detection** on the original index raster (not on a thresholded version, to avoid false edges at NoData boundaries).
3. Builds a **vegetation mask** using the loaded threshold and combines it with the edge mask (`logical AND`).
4. **Dilates** the edges (5 pixels ≈ 1 m) to create a buffer around real vegetation transitions.
5. Samples index values at edge pixels, then applies **Otsu's method** to find the optimal threshold.
6. Classifies the full raster using the Otsu threshold.
7. Exports the mask and appends stats to the CSV.

---

### VUDO-4-Clumping_Mask.ipynb

**Purpose**: Apply **clump-size filtering** to remove vegetation patches smaller than a minimum area (default 2500 pixels = 100 m²). Small patches are often false positives (isolated trees, shadows misclassified as vegetation, etc.).

**Input**:

- The index mosaic from Step 1.
- The CSV from Steps 2/3 (to read the threshold for the chosen `method`).

**Output**:

- `<city>-<procom>-<index>-<year>-<method>_clump.tif` — final vegetation mask after clump filtering.
- Appends a row to the existing CSV with the clumped mask stats.

**Key operations**:

1. Loads the threshold for the configured `method` (KDE, KMeans, or Australian).
2. Classifies the full raster into a binary 0/1 mask.
3. Processes the mask in **horizontal strips** (2000 pixels high) to bound memory usage.
4. For each strip, labels connected clumps (4-neighborhood connectivity) using `scipy.ndimage.label`.
5. Keeps only clumps larger than `min_pxl_num` (2500 pixels).
6. For clumps spanning strip boundaries, uses **overlapping strips** and a `logical OR` to avoid artificial cuts.
7. Applies a final `logical AND` with the original binary mask to remove non-green areas.
8. Restores NoData (255) at pixels that were originally NoData before exporting.

---

### VUDO-5-Clumping_Stats.ipynb

**Purpose**: Compute **size statistics** for the vegetation clumps produced by Step 4—mean, std, quartiles, min, max—in pixels and m².

**Input**:

- `<city>-<procom>-<index>-<year>-<method>_clump.tif` from Step 4.

**Output**:

- `<city>-<procom>-<index>-<year>-<method>--clump_stats.csv` — size-distribution statistics.

**Key operations**:

1. Loads the clump mask.
2. Splits it into **horizontal strips** (100 pixels high for large rasters, configurable via `tile_h`).
3. Labels clumps in each strip independently.
4. Uses a **graph** (`networkx`) to reconnect clumps that were split by strip boundaries: each edge connects an "upper" clump ID to a "lower" clump ID that touches at the border.
5. Aggregates pixel counts across reconnected components.
6. Computes summary statistics (mean, std, min, 25%, 50%, 75%, max) in pixels and m².
7. Exports as CSV.

---

## 5. Usage Notes and Caveats

### Pixel size assumptions

The pipeline assumes **20 cm/pixel** by default (`pxl_area = 0.04 m²`). The only exception is Reggio Calabria 2012 (50 cm/pixel, `pxl_area = 0.25`), handled by an `if` block in Steps 3 and 4. If you use orthophotos with a different resolution, update `pxl_area` and `min_pxl_num` accordingly.

### NoData conventions

- **Input index rasters** (Step 1 output): NoData = `-20`.
- **Output masks** (Steps 2–5): NoData = `255`.

All writers declare the NoData value on the output band, but the actual masking is done upstream on the numpy arrays.

### Memory and performance

- Step 2 samples 1% of pixels for KDE/KMeans to avoid loading the full raster into the clustering algorithm.
- Steps 4 and 5 process the raster in horizontal strips to bound peak memory usage.
- For large cities (e.g., Roma, Genova), Step 4 can take **~1 hour or more**. The notebook includes timing comments for reference.

### CRS handling

Step 1 reads the CRS of the first RGBI tile and looks it up in `CRScorrespondences`. If your raster uses a CRS not in that dictionary, you must add it before running, or you will get a `KeyError`.

### Index formulas

The pipeline supports `NDVI_red`, `NDVI_blue`, and `ENDVI`. The formulas are defined in Step 1's `evaluate_index()` function. If you need a different index (e.g., EVI, SAVI), add a branch there.

### `kde_threshold` fallback

In Step 2, if KDE finds no valid minima (e.g., for bimodal histograms that are too flat), the threshold defaults to `kde_min[-1]`, which may be `0.0` or negative. Step 3's `threshold_method = 'auto'` handles this by falling back to the KMeans threshold.

---

## 6. License and Attribution

The code in this repository is provided for research and non-commercial use. **You must respect AGEA's license terms when using their orthophotos.** As of September 2026, AGEA orthophotos are released under **CC BY 4.0**, which requires attribution to AGEA and indication of any modifications.

If you use this pipeline in academic work, please cite the Australian method paper (MDPI, 2016) and acknowledge AGEA as the orthophoto source.

---

## 7. References

- Australian adaptive pipeline method: [MDPI Remote Sensing, 2016](https://www.mdpi.com/2072-4292/8/5/386)
- AGEA GeoPortale download procedure: [Geocorsi](https://www.geocorsi.it/N1787/geoportale-agea-online-il-nuovo-servizio-per-consultare-e-scaricare-le-ortofoto-nazionali.html)
- AGEA GeoPortale license (CC BY 4.0): [GISinfrastrutture, Sept 2026](https://www.gisinfrastrutture.it/2026/09/ora-le-ortofoto-agea-sono-davvero-open/)
- KDE bandwidth rule-of-thumb: [Wikipedia](https://en.wikipedia.org/wiki/Kernel_density_estimation#A_rule-of-thumb_bandwidth_estimator)
- Canny edge detection: [scikit-image documentation](https://scikit-image.org/docs/stable/api/skimage.feature.html#skimage.feature.canny)
