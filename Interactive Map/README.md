# Interactive Lunar Map

The interactive map is the primary deliverable of this project — a self-contained HTML file that places every processed CLASS observation as a hoverable pin on a lunar albedo basemap. Hovering any point reveals the full elemental composition of that 12.5 × 12.5 km surface cell: Si-normalised ratios for O, Na, Mg, Al, Ca, Ti, Mn, and Fe, each accompanied by its propagated measurement uncertainty.

![INTERACTIVE MAP EXAMPLE](../assets/interactive_map_example.png)

The hover card shows lat/lon coordinates alongside all eight element/Si ratios with uncertainties. All of this is accessible at any point on the map, in real time, across hundreds of thousands of data points.

---

## The Engineering Problem

The raw catalog from the spectral pipeline contains well over a hundred thousand observation rows, each describing an irregularly shaped 12.5 km quadrilateral footprint. Plotting this naively — as individual SVG elements in a standard scatter plot — is not viable. The browser DOM builds one element per point, and rendering freezes long before the full dataset is on screen. Interaction is impossible.

Two interconnected problems needed solving:

1. **Geometric irregularity** — the raw footprints are quadrilaterals with four different corner coordinates each. There is no direct path from that structure to a consistent pixel coordinate system on a rectangular basemap image.
2. **Scale** — the rendered output needs to handle hundreds of thousands of simultaneous data points with interactive hover at each, without freezing the browser.

Both were solved by separating the problem into distinct stages: first normalise the data geometry, then hand the rendering entirely to the GPU.

---

## Stage 1 — Regridding Irregular Footprints onto a Uniform Grid

The quadrilateral footprints from the catalog cannot be projected pixel-by-pixel without expensive polygon intersection logic for every frame. The solution is to abandon the footprint geometry and recast the data onto a regular 0.1° × 0.1° latitude-longitude grid that maps cleanly to pixel coordinates on the basemap.

This is done using a **KD-tree spatial index** built over the catalog's observation coordinates. The full observation extent is covered with a uniform grid of query points at 0.1° spacing. The KD-tree answers the question "which catalog observation is nearest to each grid point?" for all grid points simultaneously — not by looping, but as a single vectorised batch query running in compiled code. Every grid cell inherits the elemental ratios of its nearest observation in one pass. What would take minutes in a Python loop completes in under a second.

The result is a clean rectangular grid DataFrame — one row per 0.1° cell, each carrying all eight element/Si ratios and their uncertainties — that maps directly to pixel coordinates with a linear equirectangular formula.

---

## Stage 2 — Sub-Pixel Resolution through Overlap Averaging

Where multiple orbital passes cover the same 0.1° grid cell — which happens frequently at mid-latitudes where ground tracks converge — each pass is an independent spectral measurement of the same surface region. Rather than discarding this redundancy, the pipeline detects duplicate (lat, lon) cells after regridding and averages their ratio values.

![SUBPIXEL METHODOLOGY](../assets/subpixel_methodology.png)

The right panel shows the resulting grid. Dense multi-pass coverage areas resolve compositional detail finer than the native 12.5 km footprint, because each contributing spectrum is fitted independently and the mean of independent fits carries lower uncertainty than any single measurement. This is the sub-pixel resolution enhancement: no interpolation, no upsampling — just the correct statistical treatment of overlapping real measurements.

---

## Stage 3 — Pixel Coordinate Mapping

Once the data is on a uniform lat/lon grid, converting to pixel coordinates on the basemap image is a straightforward linear transform — the full −180° to +180° longitude range maps to image width, and −90° to +90° latitude maps to image height with the y-axis flipped to match image convention. Every data point is positioned at sub-pixel precision on the basemap without any reprojection library.

---

## Stage 4 — GPU Pin-Level Rendering with WebGL

With pixel coordinates computed, the rendering backend determines everything about whether the map is usable.

Standard Plotly `Scatter` renders via SVG. The browser creates a DOM node for every single point. At 10,000 points the page starts lagging; at 100,000 it becomes unresponsive; at the scale of this dataset it does not render at all.

**`Scattergl`** routes the entire render pipeline through **WebGL** — the browser's interface to the GPU. Instead of DOM nodes, point coordinates and attributes are uploaded once as compact vertex buffers directly into GPU memory. The GPU then draws all points in parallel across its thousands of shader cores. The CPU is not involved in the draw loop. Zoom, pan, and hover interactions trigger a GPU redraw that completes in milliseconds regardless of point count.

Each observation is rendered at `size=1` — a single screen pixel, the finest spatial resolution the display allows. This is intentional: at this scale, dense coverage regions appear as filled areas reflecting actual data density, while sparse tracks from individual orbital passes remain individually visible. It is a direct visual representation of coverage, not an artificial smoothing.

| Renderer | Backend | Practical limit | Zoom/pan at scale |
|---|---|---|---|
| Scatter | SVG / CPU DOM | ~10,000 points | Freezes |
| Scattergl | WebGL / GPU | 1,000,000+ points | Real-time |

**Why hover still works at this scale:** all eight element ratios for every point are packed into the `customdata` buffer and uploaded to GPU memory at render time. When you hover a point, the browser reads directly from that buffer — no event callbacks, no server requests, no layout recalculations. The hover card appears instantly because the data is already in the right place.

---

## Data Preparation Flow

```
Raw catalog CSV
(irregular quadrilateral footprints, 9 element areas per row)
        │
        ▼
KD-tree nearest-neighbour regrid → uniform 0.1° grid
        │
        ▼
Element/Si ratio computation per cell
        │
        ▼
Overlap deduplication — mean-aggregate duplicate cells
(sub-pixel resolution enhancement)
        │
        ▼
Equirectangular lat/lon → pixel coordinate mapping
        │
        ▼
WebGL Scattergl render on lunar albedo basemap
        │
        ▼
Self-contained interactive HTML output
```

---

## What the Map Shows

- **Ratios, not raw counts** — all values are normalised to Si, which is homogeneously distributed across the lunar surface. This removes solar flux variation as a confound: a high Fe/Si ratio means Fe is genuinely enriched relative to the surface average, not that the observation happened during a strong flare.
- **Uncertainty at every point** — the hover card shows the propagated spectral overlap uncertainty alongside each ratio. Points near spectral overlap regions (e.g., where Mg and Al peaks interfere) carry higher uncertainty, visible directly in the map.
- **Coverage as a visual signal** — the density of rendered pins is a direct proxy for data coverage. Mare regions with many overlapping orbital passes appear dense; polar regions with sparse tracks appear as individual lines.

---

## Files

| File | Description |
|---|---|
| `interactive_web_map.ipynb` | Final rendering notebook — basemap + Scattergl trace |
| `CSV Preparation of interactive_web_map.ipynb` | KD-tree regridding and overlap averaging |
| `final_interactive_map_ratios_data.csv.zip` | Processed grid (ratio ± uncertainty per 0.1° cell) |
| `individual_raw_fits_data.csv.zip` | Raw catalog output from the Gaussian pipeline |
| `interactive_map_demo.html` | Rendered interactive map |
