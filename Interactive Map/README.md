# Interactive Lunar Map

The interactive map is a GPU-rendered HTML visualization that places every CLASS XRF observation as a hoverable pin on a lunar albedo basemap. Hovering a point surfaces its Si-normalized elemental ratios and measurement uncertainties — covering 8 elements (O, Na, Mg, Al, Ca, Ti, Mn, Fe) for each 12.5 × 12.5 km observation footprint.

The output is a self-contained HTML file. It is heavy (~hundreds of MB of inline data) but requires no server — open it in a browser and the GPU takes over.

---

## Why This Needed Careful Engineering

The catalog CSV contains hundreds of thousands of observation rows. A naive SVG-based scatter plot (e.g., standard Plotly Express) collapses under that load — the DOM builds one element per point and the browser freezes long before rendering completes. The two key decisions that make this work at scale are:

1. **GPU-accelerated rendering via WebGL (Scattergl)**
2. **cKDTree-based spatial regridding instead of per-row iteration**

---

## Stage 1: Spatial Regridding with cKDTree

The raw catalog rows each describe an irregular quadrilateral footprint with 4 corner lat/lon pairs. Before rendering, those footprints are mapped to a uniform 0.1° × 0.1° grid that aligns cleanly with pixel coordinates on the basemap image.

```python
from scipy.spatial import cKDTree
import numpy as np

# Build a uniform grid covering the full observation extent
latitudes  = np.arange(lat_min, lat_max, 0.1)
longitudes = np.arange(lon_min, lon_max, 0.1)
grid_points = np.array(np.meshgrid(latitudes, longitudes)).T.reshape(-1, 2)

# Index the catalog by V0 corner coordinates
tree = cKDTree(df[["V0_lat", "V0_lon"]])

# Single vectorized query — all grid points at once
distances, indices = tree.query(grid_points)

# Assign element ratios from nearest catalog row to each grid point
grid_df["Fe/Si_ratio"] = df.iloc[indices]["Fe_area"].values / df.iloc[indices]["Si_area"].values
```

**Why cKDTree instead of a loop:**

A KD-tree partitions the coordinate space into a binary search tree, reducing nearest-neighbor lookup from O(n) to O(log n) per query. More importantly, `tree.query(grid_points)` accepts the entire grid array at once. NumPy-backed batch queries run in compiled Cython code with no Python loop overhead — the full assignment for hundreds of thousands of grid points completes in under a second. A Python `for` loop over the same data would take minutes.

The result is a clean rectangular grid DataFrame with one row per 0.1° cell, each carrying all 8 element ratios and their uncertainties.

---

## Stage 2: Overlap Averaging

Where multiple catalog observations fall within the same 0.1° grid cell (overlapping footprints in high-density coverage areas), the ratio values are averaged:

```python
grouped_df = (
    df.groupby(['latitude', 'longitude'], as_index=False)
    .agg({**{col: 'mean' for col in mean_columns},
          **{col: 'mean' for col in uncertainty_columns}})
)
```

This is where the "sub-pixel resolution" comes from: overlapping observations covering the same cell contribute independently fitted spectra, and their average represents finer compositional detail than a single pass would give.

---

## Stage 3: Pixel Coordinate Mapping

Lat/lon are converted to pixel coordinates on the basemap image using a simple equirectangular projection:

```python
df['x_pixel'] = ((df['Longitude'] + 180) / 360) * img_width
df['y_pixel'] = ((df['Latitude'] - 90) / 180) * img_height
```

This maps the full −180°→+180° longitude range and −90°→+90° latitude range directly to image pixel space. The y-axis flip (−90 at bottom, +90 at top) is absorbed into the formula. Every data point is then positioned at sub-pixel accuracy relative to the basemap.

---

## Stage 4: GPU Pin-Level Rendering with Scattergl

```python
import plotly.graph_objects as go

fig.add_trace(
    go.Scattergl(
        x=df['x_pixel'],
        y=df['y_pixel'],
        mode="markers",
        marker=dict(size=1, color='blue', opacity=0.25),
        hovertemplate=(
            "Lat: %{customdata[0]:.2f}<br>"
            "Lon: %{customdata[1]:.2f}<br>"
            "Fe/Si_ratio: %{customdata[9]}<extra></extra>"
            # ... all 8 elements
        ),
        customdata=df[[
            "latitude", "longitude",
            "O/Si_ratio", "Na/Si_ratio", "Mg/Si_ratio", "Al/Si_ratio",
            "Ca/Si_ratio", "Ti/Si_ratio", "Mn/Si_ratio", "Fe/Si_ratio"
        ]].values
    )
)
```

**`Scattergl` vs `Scatter` — the critical difference:**

| | `go.Scatter` | `go.Scattergl` |
|---|---|---|
| Renderer | SVG (CPU, DOM) | WebGL (GPU, canvas) |
| Max practical points | ~10,000 before freezing | 1,000,000+ smoothly |
| Zoom/pan | Lags with large datasets | GPU re-renders in real time |
| Hover | Same | Same |

`Scattergl` delegates the entire rendering pipeline to WebGL. The browser uploads point coordinates and attributes to GPU memory as vertex buffers, and the GPU draws all points simultaneously — each one rendered as a 1×1 pixel pin. The CPU is not involved in the draw loop at all.

With `marker.size=1`, every observation footprint becomes a single screen pixel — the highest resolution the display allows. Dense coverage regions appear as filled areas; sparse coverage reveals individual measurement tracks from the satellite's orbital passes.

**`customdata` for hover without layout overhead:**

All 8 element ratios per point are packed into `customdata` at render time and stored client-side in the WebGL buffer. The `hovertemplate` reads directly from that buffer — no event callbacks, no data fetches, no layout recalculations. Hover response is instantaneous even on the full dataset.

---

## Data Preparation Flow

```
endfinal_individualfin1.csv          (raw catalog: irregular footprints, element areas)
          │
          ▼
  cKDTree nearest-neighbor regrid to 0.1° uniform grid
          │
          ▼
  Ratio computation: element_area / Si_area
          │
          ▼
  Overlap deduplication: group by (lat, lon), mean-aggregate
          │
          ▼
  final_map_interactive_updated.csv   (uniform grid, ratio ± uncertainty per cell)
          │
          ▼
  Pixel coordinate mapping (equirectangular)
          │
          ▼
  Scattergl render → interactive_map.html
```

---

## Files

| File | Description |
|---|---|
| [`interactive_web_map.ipynb`](interactive_web_map.ipynb) | Final rendering notebook (Scattergl + basemap) |
| [`CSV Preparation of interactive_web_map.ipynb`](CSV%20Preparation%20of%20interactive_web_map.ipynb) | cKDTree regridding + overlap averaging |
| [`final_interactive_map_ratios_data.csv.zip`](final_interactive_map_ratios_data.csv.zip) | Processed grid CSV (ratio ± uncertainty per cell) |
| [`individual_raw_fits_data.csv.zip`](individual_raw_fits_data.csv.zip) | Raw catalog output from Gaussian pipeline |
| [`interactive_map_demo.html`](interactive_map_demo.html) | Rendered interactive map |

---

## Running Locally

```bash
pip install plotly pandas numpy scipy Pillow
jupyter notebook interactive_web_map.ipynb
```

The HTML output requires a browser with WebGL enabled (all modern browsers do by default). For the full dataset, 8–16 GB RAM is recommended.
