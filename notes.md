# Project Notes

Running log of decisions, problems and fixes. Used for writing the Methods section later.

---

## Step 1: Feature stack (25 Sept 2026)

### Study area
- Chose **Rudraprayag district** instead of all of Uttarakhand, to keep things fast to iterate on.
- Reasons: very landslide-prone, on the Kedarnath route, already studied in published papers I can compare against.
- Boundary from **FAO GAUL 2025 level 2** (`FAO/GAUL/2025/level2`).
  - First tried GAUL 2015, but it's deprecated and didn't find the district by name (it uses old names like "Uttaranchal").
  - Searching with `stringContains("GAUL2_NAME", "Rudra")` so small spelling differences don't break it.
  - Official Rudraprayag boundary found with GAUL 2025.

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
  Switched to Esri World Imagery as the basemap, which also suits a terrain project better.
- GAUL 2015 deprecation warning, fixed by moving to GAUL 2025.
- Land cover panel first used a continuous colorbar, which is wrong for categorical data.
  Replaced with official ESA WorldCover colours and a labelled legend.

### Figure 2 made
- 6-panel conditioning factors figure, read from the exported GeoTIFF at 60 m (half resolution) for speed.
- Saved as PNG (300 dpi) and PDF in Drive under `landslide_project/figures/`.
- Pixels outside the district are exported as 0, masked as NaN using elevation == 0 (no real pixel is at 0 m here).
- Added to GitHub in `figures/` and shown in the README.

### First observations (for Results / Discussion)
- Elevation ranges from ~700 m in the southern river valleys to ~7,000 m in the north (Kedarnath peaks).
- Most of the district has slopes between 25 and 45 degrees.
- Dense vegetation (high NDVI) through the middle; snow, ice and bare rock in the north.
- **Idea for Step 3:** the high-altitude snow/ice zone behaves differently and has few settlements.
  Consider excluding permanent snow and ice (WorldCover class 70) from the model, as many studies do.

---

## Step 2: Landslide inventory (started 25 Sept 2026)

### Data access problems
- **GSI Bhukosh portal** (bhukosh.gsi.gov.in) wasn't loading (tried 25 Sept 2026). Government portals go down often; retrying occasionally.
- **NASA COOLR** (Cooperative Open Online Landslide Repository) was the backup source:
  - Events layers (points and polygons) on gis.earthdata.nasa.gov returned 404, probably after NASA's Earthdata GIS system upgrade.
  - maps.nccs.nasa.gov was unreachable from Colab ("Network is unreachable").
  - Reports Points layer: the layer info page opened, but every query returned an nginx 404,
    both from Colab and from my own browser. So the query service is down, not just blocked for Colab.
- Conclusion: no existing inventory was accessible, so I built my own.

### First attempt: manual hunting (abandoned)
- Tried finding scars by hand in the Earth Engine Code Editor
  (repository `users/makkergauri/landslide_uttarakhand`, script `map_landslides`).
- Found the first landslide (debris track near Sonprayag–Gaurikund on the Kedarnath route),
  but it was far too slow to reach 100+ points.

### Current method: semi-automatic inventory (candidate detection + manual verification)
- Script `review_candidates` in `users/makkergauri/landslide_uttarakhand`.
  1. **Automatic candidates:** pixels with NDVI < 0.2 (Sentinel-2, Oct–Dec 2025), slope > 25°,
     not snow/ice or water (ESA WorldCover 70, 80), elevation < 3500 m (above the treeline bare rock is natural).
     Converted to patches with `reduceToVectors` at 20 m; kept patches of 0.2–20 ha.
     Result: **330 candidate patches** in the district.
  2. **Manual verification:** random sample of 300 patches (seed 42), each reviewed on high-resolution
     Google imagery at zoom 17 and labelled yes / no / unsure. May extend to all 330.
- Landslide points = centroid of each patch labelled "yes".
- "No" patches are also kept: they are verified non-landslides and can be used as negative examples.
- Exported as GeoJSON to Drive (`landslide_review_rudraprayag`).

### Review sessions
- **25 Sept 2026, session 1:** first 10 candidates → 3 yes, 6 no, 1 unsure (precision ≈ 33%). Saved and exported.
- **25 Sept 2026, session 2:** page reloaded and unsaved work after candidate 10 was lost.
  Restarted from candidate 11 using the saved first 10. Now saving every 10 candidates.
- Final results: in progress (reviewed __, yes __, no __, unsure __).

### Decision rules I settled on while reviewing
- stream beds and gullies (grey strips in valley bottoms, same width all the way, joining other channels) → no
- buildings, construction sites, roads, riverbeds, white water → no
- plain road cut with no debris on the road and no bowl-shaped scar → no / unsure
- rock cliffs and ridge edges in shadow → no
- bright, clean bare patch cut into forest, sharp edge, spreading downhill → yes
- bare slope collapsing into a stream (streamside slide) → yes
- huge boulder fan much wider than a normal stream, fed by a steep channel (debris-flow deposit) → yes
- dull brown slope with scattered bushes, or high alpine brown terrain → unsure
- same landslide as an earlier candidate → unsure (avoids counting one landslide twice)
- zoom out 2 levels when the outline sits inside a bigger bare area
- more than 30 seconds without a clear answer → unsure

### Limitations (for the paper)
- No field verification; single interpreter (me), so some subjectivity.
- Only landslides that were still bare in late 2025 and larger than 0.2 ha can be found,
  so older revegetated and very small landslides are under-represented.
- Landslide points are patch centroids, not initiation points (top of the scar).
- **Possible circularity:** candidates were found using low NDVI and steep slope, and NDVI and slope are also
  model features. This could make those two features look more important than they are.
  To check in Step 3: compare the model with and without NDVI.
- If Bhukosh comes back, use GSI points as an independent check of my inventory.

---

## To do next
- [x] Step 1: feature stack exported
- [x] Figure 2 generated and added to GitHub and README
- [x] Step 2: review tool built, first 10 candidates saved
- [ ] Step 2: finish reviewing all candidates, save, run the export task
- [ ] Step 2: Colab notebook `02_landslide_inventory`: load results, remove duplicates,
      generate "no landslide" points, make Figure 3
- [ ] Step 3: random forest + spatial cross-validation + NDVI circularity check
- [ ] Keep retrying Bhukosh

## Figures planned for the paper
1. Study area map (location within India and Uttarakhand)
2. Conditioning factors panel (6 layers) ← done
3. Landslide inventory map
4. Model evaluation (ROC curve, feature importance)
5. Final susceptibility map
6. Rainfall and monsoon risk analysis

---

## Data citations
(Check each dataset's Earth Engine catalog page for the exact citation format before the paper.)
- **SRTM:** Farr, T. G. et al. (2007). The Shuttle Radar Topography Mission. *Reviews of Geophysics*, 45.
- **Sentinel-2:** Contains modified Copernicus Sentinel data (2024–2025), ESA.
- **ESA WorldCover:** Zanaga, D. et al. (2022). ESA WorldCover 10 m 2021 v200. doi:10.5281/zenodo.7254221
- **HydroSHEDS Free Flowing Rivers:** Grill, G. et al. (2019). Mapping the world's free-flowing rivers. *Nature*, 569, 215–221.
- **FAO GAUL:** FAO (2025). Global Administrative Unit Layers (GAUL). Licence: CC-BY-4.0.
- **NDVI:** Rouse, J. W. et al. (1974). Monitoring vegetation systems in the Great Plains with ERTS.

### Only if GSI or NASA data ends up being used
- **GSI:** Geological Survey of India, Bhukosh landslide inventory (bhukosh.gsi.gov.in), accessed ____.
- **COOLR:** Juang, C. S., Stanley, T. A., & Kirschbaum, D. B. (2019). Using citizen science to expand the global map
  of landslides: Introducing the Cooperative Open Online Landslide Repository (COOLR). *PLOS ONE*, 14(7), e0218657.
- **NASA GLC:** Kirschbaum, D. B., Stanley, T., & Zhou, Y. (2015). Spatial and temporal analysis of a global landslide catalog.
  *Geomorphology*, 249, 4–15.
