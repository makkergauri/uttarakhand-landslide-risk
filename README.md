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

## Roadmap

- [x] Step 1: Build feature stack in Google Earth Engine (`01_build_features.ipynb`)
- [x] Figure 2: conditioning factors
- [x] Step 2: Landslide inventory: 300 candidates reviewed, 42 landslides verified
- [ ] Step 3: Train random forest + spatial cross-validation
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
| `figures/` | Figures for the paper |
| `notes.md` | Running log of every decision, problem and fix |

## Run it

1. Get a free [Google Earth Engine](https://earthengine.google.com/) account (noncommercial use).
2. Open `01_build_features.ipynb` and click the **Open in Colab** button.
3. Replace the project ID in the first cell with your own Earth Engine project.

## Acknowledgements

Code scaffolding and project planning developed with help from Claude (Anthropic).
