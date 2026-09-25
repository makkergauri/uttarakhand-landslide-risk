# Project Notes

Running log of decisions, problems and fixes. Used for writing the Methods section later.

---

## Step 1: Feature stack (Sept 2026)

### Study area
- Chose **Rudraprayag district** instead of all of Uttarakhand, to keep things fast to iterate on.
- Reasons: very landslide-prone, on the Kedarnath route, already studied in published papers I can compare against.
- Boundary from **FAO GAUL 2025 level 2** (`FAO/GAUL/2025/level2`).
  - First tried GAUL 2015, but it's deprecated and didn't find the district by name (it uses old names like "Uttaranchal").
  - Searching with `stringContains("GAUL2_NAME", "Rudra")` so small spelling differences don't break it.

### Conditioning factors
| Factor | Dataset | Why it matters |
|---|---|---|
| Elevation | NASA SRTM 30 m (`USGS/SRTMGL1_003`) | base terrain |
| Slope | derived from SRTM | steeper = more likely to fail, usually the strongest factor |
| Aspect | derived from SRTM | direction a slope faces affects sunlight, moisture, vegetation |
| NDVI | Sentinel-2 SR Harmonized | roots stabilise soil, bare slopes are weaker |
| Land cover | ESA WorldCover v200 (10 m) | forest vs cropland vs built-up vs bare rock |
| Distance to river | WWF HydroSHEDS Free Flowing Rivers | rivers erode the base of slopes (toe erosion) |

### Processing choices
- **NDVI date range: Oct 2024 to Mar 2025**, because monsoon (Jun to Sep) imagery is almost all cloud.
  - Filtered to scenes with less than 20% cloud, then took the **median** composite to remove leftover clouds.
  - NDVI = (B8 − B4) / (B8 + B4), i.e. (NIR − Red) / (NIR + Red).
- **Rivers:** buffered the study area by 20 km before selecting rivers, so rivers just outside the boundary still count for distance.
- **Export:** 30 m resolution, **EPSG:32644 (UTM zone 44N)** so units are metres. Exported to Google Drive as `rudraprayag_features.tif`.

### Problems and fixes
- geemap's default OpenStreetMap background tiles were blocked in Colab (403 "Access blocked").
  Switched the basemap to Google `HYBRID` satellite imagery, which also suits a terrain project better.
- GAUL 2015 deprecation warning, fixed by moving to GAUL 2025.

### Figures planned for the paper
1. Study area map (location within India and Uttarakhand)
2. Conditioning factors panel (6 layers) ← made in Step 1
3. Landslide inventory map
4. Model evaluation (ROC curve, feature importance)
5. Final susceptibility map
6. Rainfall and monsoon risk analysis

### To do next
- [ ] Confirm export finished and Figure 2 is generated
- [ ] Step 2: get landslide inventory from GSI Bhukosh

---

## Data citations
(Check each dataset's Earth Engine catalog page for the exact citation format before the paper.)
- **SRTM:** Farr, T. G. et al. (2007). The Shuttle Radar Topography Mission. *Reviews of Geophysics*, 45.
- **Sentinel-2:** Contains modified Copernicus Sentinel data (2024–2025), ESA.
- **ESA WorldCover:** Zanaga, D. et al. (2022). ESA WorldCover 10 m 2021 v200. doi:10.5281/zenodo.7254221
- **HydroSHEDS Free Flowing Rivers:** Grill, G. et al. (2019). Mapping the world's free-flowing rivers. *Nature*, 569, 215–221.
- **FAO GAUL:** FAO (2025). Global Administrative Unit Layers (GAUL). Licence: CC-BY-4.0.
- **NDVI:** Rouse, J. W. et al. (1974). Monitoring vegetation systems in the Great Plains with ERTS.
