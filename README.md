# High-Resolution Lunar Elemental Mapping

**Inter IIT Tech Meet 13.0 — ISRO Problem Statement 4 | IIT Bhubaneswar**

---

## Project Overview

This project processes orbital X-ray fluorescence (XRF) data from the **Chandrayaan-2 CLASS** (Large Area Soft X-ray Spectrometer) instrument to produce compositional maps of the lunar surface. The core challenge is extracting clean elemental signals from noisy low-count FITS spectra, converting them to elemental ratios, and serving the complete dataset as a GPU-accelerated interactive visualization.

The pipeline moves through three stages: **spectral catalog construction → elemental base maps → interactive geochemical map**.

---

## Modules

### 1. XRF Spectral Catalog — [`Elemental abundances/`](Elemental%20abundances/)

Raw FITS files from ISRO's Pradan archive (2021–2024) arrive as 2048-channel photon count arrays, one file per ~0.5 s observation window. Every observed spectrum is contaminated by background and is only useful during active solar flare illumination. The catalog pipeline:

- **Solar flare gating** — cross-references XSM (Solar X-ray Monitor) flare records to discard quiescent spectra. Only flare-illuminated observations carry sufficient flux to produce detectable XRF lines.
- **Background estimation** — a moving average over the low-energy continuum, updated with an exponential smoothing coefficient (β = 0.05), replaces a static background. This is critical because the background drifts across an orbit.
- **Batch co-addition** — 8 consecutive FITS files are co-added into a single spectrum. This is a deliberate SNR trade-off: individual files have too few counts for a reliable Gaussian fit, but 8-file batches average ~12.5 km spatial footprints, which matches the CLASS native resolution anyway.
- **Gaussian decomposition** — each co-added spectrum is decomposed into 10 Gaussians, one per element (O, Fe×2, Na, Mg, Al, Si, Ca, Ti, Mn). The fit handles spectral overlap analytically: the area of the overlapping tail of each Gaussian with the Si reference peak is computed and subtracted before forming element/Si ratios.
- **Oxygen edge case** — the O K-alpha line sits at channels 37–38, within the rising noise floor. A channel-mirroring heuristic detects and removes the instrumental turn-on artifact before fitting.

Output per batch: 8 spatial coordinates (4 corner lat/lons), 9 elemental Gaussian areas, 8 element/Si uncertainty values.

---

### 2. Elemental Base Maps — [`Lunar Basemaps/`](Lunar%20Basemaps/)

The catalog rows are projected onto a lunar albedo basemap. Each 12.5 × 12.5 km observation footprint is drawn as a color-coded bounding box (4-corner polygon), blended onto the basemap via alpha compositing. The viridis colormap normalizes each element independently so relative variation is visible.

**Key geochemical results visible in the maps:**

| Pattern | Interpretation |
|---|---|
| Al anti-correlated with Mg and Fe | Feldspathic highlands (Al-rich anorthosite) vs. mafic mare basalts |
| Mg + Fe concentrated in Oceanus Procellarum | Volcanic resurfacing, thin crust, Fe-Ti basalt |
| Si spatially uniform | Consistent with use as normalization reference |
| Ti elevated in select maria | Ti-rich ilmenite basalts, consistent with Apollo sample data |

**Elemental coverage maps at two opacity thresholds (0.4 and 0.7):**

| Element | 0.4 opacity | 0.7 opacity |
|---------|------------|------------|
| **Fe** | ![Fe 0.4](Lunar%20Basemaps/final_images/Fe_0.4._f.png) | ![Fe 0.7](Lunar%20Basemaps/final_images/Fe_0.7_f.png) |
| **Mg** | ![Mg 0.4](Lunar%20Basemaps/final_images/Mg_0.4_f.png) | ![Mg 0.7](Lunar%20Basemaps/final_images/Mg_0.7_f.png) |
| **Al** | — | ![Al 0.7](Lunar%20Basemaps/final_images/Al_0.7_f.png) |
| **Ca** | ![Ca 0.4](Lunar%20Basemaps/final_images/Ca_0.4_f.png) | ![Ca 0.7](Lunar%20Basemaps/final_images/Ca_0.7_f.png) |
| **Ti** | ![Ti 0.4](Lunar%20Basemaps/final_images/Ti_0.4_f.png) | ![Ti 0.7](Lunar%20Basemaps/final_images/Ti_0.7_f.png) |
| **Si** | ![Si 0.4](Lunar%20Basemaps/final_images/Si_0.4_f.png) | ![Si 0.7](Lunar%20Basemaps/final_images/Si_0.7_f.png) |

---

### 3. Interactive Lunar Map — [`Interactive Map/`](Interactive%20Map/)

The centerpiece of the project. A single HTML file that overlays all XRF observations on a lunar albedo basemap, with per-point hover showing element/Si ratios and uncertainties for every 12.5 × 12.5 km footprint.

This is covered in detail in the next section.

---

## Pipeline Summary

```
Raw FITS (Pradan archive)
       │
       ▼
 Solar flare gate (XSM cross-reference)
       │
       ▼
 Co-add 8 files → single SNR-boosted spectrum
       │
       ▼
 Gaussian decomposition (10 elements)
 + overlap correction + uncertainty
       │
       ▼
 Per-observation CSV: lat/lon + 9 element areas + 8 uncertainties
       │
       ├──► Heatmap basemaps (static PNG)
       │
       └──► cKDTree regridding → Interactive map (GPU-rendered HTML)
```

---

## Technical Stack

| Layer | Tools |
|---|---|
| FITS I/O | `astropy.io.fits` |
| Signal processing | `numpy`, `scipy.optimize.curve_fit` |
| Spatial indexing | `scipy.spatial.cKDTree` |
| Image compositing | `opencv-python`, `Pillow`, `rasterio` |
| Visualization | `plotly` (Scattergl / WebGL), `matplotlib` |

---

## Repository Contents

```
├── Elemental abundances/    # FITS catalog pipeline notebooks
├── Interactive Map/         # GPU-rendered interactive map + data prep
├── Lunar Basemaps/          # Elemental heatmap notebooks + PNG outputs
├── Report and Journal/      # Full technical report
└── ISRO_PS4_Presentation.pdf
```

---

**IIT Bhubaneswar | Inter IIT Tech Meet 13.0 | November 2024**
