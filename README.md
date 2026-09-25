# Landslide Risk Mapping in Uttarakhand 🏔️

> 🚧 **In progress.** Built in the open.

Every monsoon, landslides in Uttarakhand block highways, cut off villages and cost lives.
This project uses **free satellite data and machine learning** to map *where* slopes are
most likely to fail in **Rudraprayag district**, and later, *when* heavy rainfall pushes that risk up.

![Landslide conditioning factors, Rudraprayag](figures/fig2_conditioning_factors.png)

## Why this matters

Rudraprayag sits on the route to Kedarnath, one of the busiest pilgrimage roads in India,
in terrain that is steep, young and heavily cut by rivers and road construction.
Knowing which slopes are most vulnerable helps decide where to monitor, reinforce, or warn.

## Approach

1. **Features:** for every 30 m pixel, compute factors that influence slope stability:
   elevation, slope, aspect, vegetation (NDVI), land cover, and distance to rivers.
2. **Landslide inventory:** existing inventories (GSI Bhukosh, NASA COOLR) weren't accessible,
   so I built my own with a **semi-automatic method**:
   - the code finds bare patches on steep slopes (possible landslide scars) from Sentinel-2 imagery,
   - I review each candidate on high-resolution imagery with a custom Earth Engine tool
     and label it landslide / not a landslide / unsure.
3. **Model:** a random forest learns which combinations of factors are associated with landslides.
4. **Honest evaluation:** spatial cross-validation (training and testing on *separate areas*),
   because random splits overstate accuracy when neighbouring pixels are nearly identical.
5. **Rainfall:** add satellite rainfall (NASA GPM) to see how risk changes through the monsoon.
6. **Map:** an interactive susceptibility map anyone can explore.

## Figure 3: Landslide inventory

![Landslide inventory, Rudraprayag](figures/fig3_landslide_inventory.png)

No usable landslide inventory was reachable for Rudraprayag, so I built a preliminary one:

- **Candidates:** Sentinel-2 (Oct–Dec 2025) pixels with NDVI < 0.2, slope > 25°, below 3,500 m,
  not snow/ice or water, grouped into patches of 0.2–20 ha → **330 candidate patches**.
- **Manual review:** 300 randomly sampled candidates checked one by one on high-resolution
  satellite imagery → **42 landslides, 190 not landslides, 68 unsure**.
- **Deduplication:** landslide points within 200 m of each other merged → **38 landslides**.
- **No-landslide points:** 38 random points in the same terrain (below 3,500 m, not snow/water),
  at least 500 m from any landslide.

Grey shading shows elevation (darker = higher).

**Limitations:** small inventory from a single interpreter, no field verification, and only
landslides still bare in late 2025 and larger than 0.2 ha could be found. Landslides cluster
in the north (upper Mandakini valley), so the model could partly learn *location* instead of
slope stability. Spatial cross-validation in Step 3 is designed to catch this.

## Roadmap

- [x] Step 1: Build feature stack in Google Earth Engine (`01_build_features.ipynb`)
- [x] Figure 2: conditioning factors
- [x] Step 2: Landslide inventory: 300 candidates reviewed, 38 landslides after deduplication (`02_landslide_inventory.ipynb`)
- [x] Figure 3: landslide inventory map
- [ ] Step 3: Train random forest + spatial cross-validation ← **in progress**
- [ ] Step 4: Add monsoon rainfall
- [ ] Step 5: Interactive map on GitHub Pages
- [ ] Step 6: Technical write-up and comparison with published studies

## Data sources

| Factor | Dataset |
|---|---|
| Elevation, slope, aspect | NASA SRTM 30 m |
| Vegetation (NDVI) | Copernicus Sentinel-2 |
| Land cover | ESA WorldCover 10 m |
| Rivers | WWF HydroSHEDS |
| District boundary | FAO GAUL 2025 |
| Landslide inventory | My own, from Sentinel-2 candidates verified on high-resolution imagery |
| Rainfall (planned) | NASA GPM IMERG |

## Repository structure

| File | What it does |
|---|---|
| `01_build_features.ipynb` | Builds the 6 conditioning factors in Earth Engine and makes Figure 2 |
| `02_landslide_inventory.ipynb` | Removes duplicate landslides, samples no-landslide points, makes Figure 3 |
| `data/training_points_wgs84.geojson` | 76 training points (label 1 = landslide, 0 = no landslide), lat/lon (EPSG:4326) |
| `data/landslide_review_rudraprayag.geojson` | All 299 saved review decisions (yes / no / unsure) with candidate ID and patch area |
| `figures/` | Figures for the paper |
| `notes.md` | Running log of every decision, problem and fix |

## Run it

1. Get a free [Google Earth Engine](https://earthengine.google.com/) account (noncommercial use).
2. Open `01_build_features.ipynb` and click the **Open in Colab** button.
3. Replace the project ID in the first cell with your own Earth Engine project.
4. For `02_landslide_inventory.ipynb`: put `rudraprayag_features.tif` (from notebook 01) and
   `data/landslide_review_rudraprayag.geojson` in a Google Drive folder called `landslide_project/`,
   then run the notebook in Colab.

## Data citations

- Farr, T. G. et al. (2007). The Shuttle Radar Topography Mission. *Reviews of Geophysics*, 45.
- Contains modified Copernicus Sentinel data (2024–2025), processed by ESA.
- Zanaga, D. et al. (2022). ESA WorldCover 10 m 2021 v200. doi:10.5281/zenodo.7254221
- Grill, G. et al. (2019). Mapping the world's free-flowing rivers. *Nature*, 569, 215–221.
- FAO (2025). Global Administrative Unit Layers (GAUL) 2025. CC-BY-4.0.
- Rouse, J. W. et al. (1974). Monitoring vegetation systems in the Great Plains with ERTS (NDVI).

## Acknowledgements

Code scaffolding and project planning developed with help from Claude (Anthropic).
