# notes.md: Landslide susceptibility mapping, Rudraprayag

## Step 1: Feature stack (notebook 01_build_features)
- Boundary: FAO/GAUL/2025/level2, ISO3_CODE == "IND", GAUL2_NAME contains "Rudra". (GAUL 2015 is deprecated and uses old names like "Uttaranchal".)
- Elevation, slope, aspect: SRTM USGS/SRTMGL1_003.
- NDVI: COPERNICUS/S2_SR_HARMONIZED, Oct 2024–Mar 2025, <20% cloud, median, (B8−B4)/(B8+B4).
- Land cover: ESA/WorldCover/v200.
- Distance to river: WWF/HydroSHEDS/v1/FreeFlowingRivers, rivers within AOI buffered 20 km, distance(searchRadius=20000).
- Export: rudraprayag_features.tif, bands 1 elevation, 2 slope, 3 aspect, 4 ndvi, 5 landcover, 6 dist_river. 30 m, EPSG:32644, outside district = 0.
- geemap: OpenStreetMap tiles blocked in Colab (403), used Esri World Imagery / HYBRID.
- Figure 2: 6 panels, read at 60 m, outside masked via elevation == 0, official WorldCover colours, scale bars, north arrow, PNG 300 dpi + PDF.
- Observations: elevation ~700 m (south) to ~7,000 m (north); most slopes 25–45°; dense vegetation mid-district; snow/ice/rock in north.

## Step 2: Landslide inventory
### Data access failures
- GSI Bhukosh (bhukosh.gsi.gov.in): not loading. Retry later; if it works, use as independent validation.
- NASA COOLR: event layers on gis.earthdata.nasa.gov 404; maps.nccs.nasa.gov unreachable; COOLR_Reports_Points query nginx 404 from Colab and browser.
- Manual scar hunting: found one landslide near Sonprayag–Gaurikund, far too slow.

### Semi-automatic inventory (Code Editor script review_candidates)
- Candidates: Sentinel-2 Oct–Dec 2025 NDVI < 0.2 AND slope > 25° AND not WorldCover 70/80 AND elevation < 3500 m. reduceToVectors at 20 m, patches 0.2–20 ha → 330 candidates.
- Random sample of 300 (randomColumn seed 42), reviewed at zoom 17 on Google satellite imagery (yes / no / unsure / back / save + backup box).
- Result: 300 reviewed → 42 yes, 190 no, 68 unsure (precision ≈ 18%). Drive export has 299 of 300 (last "unsure" not saved, no effect).
- Decision rules: stream beds/gullies, buildings, construction, roads, riverbeds, white water, cliffs in shadow → no; plain road cuts → no/unsure; bright clean bare patch cut into forest with sharp edge spreading downhill, streamside slides, boulder fans fed by steep channels (debris flows) → yes; dull brown bushy or alpine brown slopes, duplicates → unsure; zoom out 2 levels when in doubt; >30 s undecided → unsure.
- Lessons: page reloads lost ~130 decisions once; top Run button restarts the tool; unsaved export tasks and Console vanish on reload. Fixed with backup box + Google Doc backups + Save → Tasks → RUN.

### Notebook 02_landslide_inventory
- Dedupe: kept one "yes" point within 200 m → 42 → 38 landslides (99 and 108 duplicate 13; 167 duplicates 88; 179 duplicates 143).
- No-landslide points: random pixels (seed 42), elevation < 3500 m, not WorldCover 70/80, ≥ 500 m from any landslide, 1:1 → 38 points.
- Confirmed counts (2026-09-25): 38 landslides, 38 no-landslide, 76 total.
- Saved training_points.geojson (EPSG:32644, label 1 = landslide, 0 = no landslide).
- Figure 3: grey elevation (darker = higher), red triangles = landslides, black dots = no-landslide, legend with n, scale bar, north arrow, PNG 300 dpi + PDF. Added district outline because low southern valleys blended into the white background.

### Observed spatial pattern (Figure 3)
- Landslides cluster in the north (upper Mandakini valley); no-landslide points are more spread and more common in the south, because more of the area below 3,500 m is in the south.
- Possible causes: genuine concentration along the Kedarnath route, and/or detection bias (bare patches easier to spot in sparser northern terrain).
- Risk: the model may learn location/elevation rather than slope stability. Spatial CV in Step 3 should expose this; discuss in the paper.

### Known limitations
- No field verification; single interpreter.
- Only landslides still bare in late 2025 and > 0.2 ha detectable.
- Small inventory (38) → preliminary.
- Points are patch centroids, not initiation points.
- Circularity: candidates selected by low NDVI + steep slope, which are also model features → compare model with and without NDVI in Step 3.
- Options to grow inventory: review remaining 30 candidates (change limit to 400), loosen candidate filter, GSI/ULMMC data.
