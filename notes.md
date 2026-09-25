# Landslide susceptibility mapping, Rudraprayag

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

### Publishing (2026-09-25)
- README updated with Figure 3 section, inventory method, limitations (incl. north clustering), data citations.
- GitHub's GeoJSON preview showed no points: training_points.geojson is in UTM (EPSG:32644), but GitHub and the GeoJSON standard expect lat/lon. Published a lat/lon copy instead (data/training_points_wgs84.geojson, EPSG:4326). Drive copy stays in UTM for the notebooks.
- Published data/landslide_review_rudraprayag.geojson (already lat/lon, exported from Earth Engine).
- Moved published data files into data/ (renamed via GitHub's edit → type "data/" before the file name).

## Step 3: Model (notebook 03_model)

### Sampling features at the training points (Cell 2)
- Sampled the 6 feature bands at all training points with rasterio `src.sample` (points reprojected to the raster CRS first).
- Sanity checks: point count, label balance, no points outside the district (elevation 0).
- Looked at mean elevation, slope, NDVI, distance to river by label, and a land-cover crosstab, before any modelling.
- Aspect and land cover not averaged (circular and categorical); handled separately.
- Sanity check results: 76 points (38/38), 0 outside district.
- Mean by label (landslide vs no landslide): elevation 2506 vs 2044 m; slope 37.2 vs 33.4°; NDVI 0.16 vs 0.62; dist_river 317 vs 1080 m.
- Land cover (landslide / no landslide): tree 7/27, grass 9/10, built-up 0/1, bare/sparse 21/0, moss/lichen 1/0.

### Circularity and choice of features
- NDVI and land cover are nearly perfect separators because candidates were selected by NDVI < 0.2 and WorldCover maps fresh scars as bare. They describe the landslide itself, not the pre-failure slope → circularity confirmed.
- Conditioning factors should describe the slope before failure. Elevation, slope, aspect and distance to river barely change when a slope fails, so they are fair predictors.
- Decision: main model = terrain-only (elevation, slope, aspect sin/cos, dist_river). Comparison model = terrain + NDVI + land cover, to show how much the circular features inflate performance.
- Distance to river differs strongly; plausible (toe erosion) but may partly reflect the "streamside slides = yes" review rule.
- Slope differs only ~4°, because random points in Rudraprayag are already steep.

### Feature preparation and spatial blocks (Cell 3)
- Aspect → sin/cos (circular: 359° and 1° are both north).
- Land cover grouped (tree 10 / grass 30 / bare 60 / other) and one-hot encoded; used only in the comparison model.
- Spatial blocks: k-means (k = 5, random_state 42) on point coordinates; spatial CV holds out one block at a time.
- Block composition: north blocks mostly landslides, middle block mixed, south blocks almost all no-landslide → out-of-fold predictions pooled across held-out blocks before computing one AUC.

### First models: random CV vs spatial CV (Cell 4)
- Model: RandomForestClassifier, 300 trees, min_samples_leaf = 2.
- Evaluation: random 5-fold CV (stratified) vs spatial CV (leave-one-block-out), each repeated 10 times (seeds 0–9; spatial blocks re-drawn with k-means each repeat). Report mean ± std AUC.
- Feature sets: terrain-only (main), full = terrain + NDVI + land cover (circular comparison), coordinates-only (x, y) as a location baseline to test whether the model just learns geography.
- Results (AUC mean ± std over 10 repeats):
  - terrain: random CV 0.841 ± 0.017, spatial CV 0.852 ± 0.004
  - full (circular): random 0.946 ± 0.005, spatial 0.947 ± 0.004
  - coordinates only: random 0.791 ± 0.022, spatial 0.829 ± 0.007
- Unexpected: spatial CV ≥ random CV. Cause: blocks differ strongly in label mix (north mostly landslides, south mostly not), so pooled AUC across held-out blocks mostly rewards separating regions, not discrimination within an area. Pooled spatial AUC here is NOT a fully honest estimate.
- Key finding: terrain (0.852) only slightly beats coordinates-only (0.829) → much of the model's skill may be location, not slope stability.
- The std values only reflect block layout; they don't capture uncertainty from the small sample (76 points). Needs a proper uncertainty estimate later (e.g. bootstrap).

### Feature diagnostics (Cell 5)
- Method: single-feature and drop-one-feature spatial CV (pooled AUC, 10 repeats); aspect sin + cos treated as one feature.
- Results: only dist_river 0.792; only elevation 0.497; only slope 0.368; only aspect 0.458. Without dist_river 0.591; without elevation 0.784; without slope 0.833; without aspect 0.896 (all terrain 0.852; coordinates only 0.829).
- Correlation with northing: elevation 0.71, slope 0.12, dist_river −0.48.
- Interpretation: distance to river carries most of the signal. Plausible (toe erosion) but may partly reflect roads (main roads, incl. the Kedarnath route, follow the river valleys; no road layer used) and the "streamside = yes" review rule. Moderately tied to location (r = −0.48 with northing).
- Elevation alone ≈ 0.5 (my expectation that it would act as a location proxy was wrong); it only helps in combination with other features.
- Single-feature AUCs below 0.5 (slope, aspect) reveal a flaw: with label-imbalanced blocks, leave-one-block-out shifts the training base rate opposite to the held-out block's, so pooled AUC is biased (up for location-like features, down for uninformative ones). Pooled spatial CV on the original points is not trustworthy.
- "Without aspect" > all features, but dropping aspect based on this would be tuning to the same 76 points → noted, not acted on.

### Matched no-landslide sampling (Cell 6)
- Root problem: landslides and no-landslide points come from different parts of the district → model can take location shortcuts, and spatial CV blocks are label-imbalanced.
- Fix: for each landslide, one random valid pixel (same terrain mask: elevation < 3500 m, not WorldCover 70/80) within 3 km (fallback 5/10 km), > 500 m from any landslide, seed 42. Pool of 200,000 random valid pixels.
- Saved as training_points_matched.geojson (EPSG:32644). Original training_points.geojson kept; its results are reported as the naive-sampling comparison.
- Trade-off: the model now answers a harder, local question (failing slope vs stable slope in the same valley), so a lower AUC is expected and more honest.
- Matched set results: all 38 landslides found a stable partner within 3 km (no fallback needed).
- Means (landslide vs matched stable): elevation 2506 vs 2649 m; slope 37.2 vs 34.6°; NDVI 0.16 vs 0.59; dist_river 317 vs 687 m.
- Elevation gap reversed (was +462 m, now −142 m) → regional height difference removed. Dist_river gap halved (763 → 370 m) but remains → part location, part local signal. Slope gap barely changed (~3°).
- Matched map: stable points now sit next to landslides; south of the district has almost no training points → susceptibility map there will be extrapolation (flag in Figure 5).

### Models on matched points (Cell 7)
- Same 3 feature sets, same RF settings, 10 repeats. Spatial CV now reports pooled AUC and within-block AUC (AUC computed inside each held-out block, averaged over blocks containing both labels).
- Coordinates-only model used as the sanity check: ≈ 0.5 means the location shortcut is removed.
- Results (mean ± std over 10 repeats):
  - coordinates only: random 0.400 ± 0.040; spatial pooled 0.485 ± 0.028; within-block 0.386 ± 0.026
  - full (circular): random 0.942 ± 0.007; spatial pooled 0.928 ± 0.005; within-block 0.935 ± 0.004
  - terrain: random 0.669 ± 0.033; spatial pooled 0.623 ± 0.023; within-block 0.721 ± 0.048
- Location shortcut removed: coordinates-only ≤ 0.5. Below 0.5 is an artefact of pairing (each landslide's nearest training neighbour is often its own stable partner), not hidden skill.
- Terrain has a real local signal: within-block AUC ≈ 0.72. This is the headline metric (it matches the question: failing vs stable slope in the same area). Report ~0.62–0.72 across evaluation types; std understates uncertainty (~15 points per block).
- Circular model still ≈ 0.93–0.94 even with matched points → bare ground marks the scar; NDVI/land cover excluded as predictors.
- Headline: naive sampling 0.85 → matched, terrain-only, within-block 0.72; circular features 0.95. Many published studies report 0.85–0.95 with random splits/district-wide background points → not directly comparable; don't claim they're wrong.

### Feature diagnostics on matched points (Cell 8)
- Within-block AUC, 10 repeats: only slope 0.660 ± 0.065; only dist_river 0.669 ± 0.041; only elevation 0.563 ± 0.021; only aspect 0.422 ± 0.006.
- Without aspect 0.737 ± 0.050; without dist_river 0.716 ± 0.016; without elevation 0.520 ± 0.104; without slope 0.704 ± 0.044 (all terrain 0.721).
- Slope and distance to river carry the local signal; elevation weak; aspect none.
- Slope caveat: candidate filter required slope > 25° (landslides guaranteed steep, stable points not) → possible partial circularity. To check: compare using only points with slope > 25°.
- Dist_river caveat: roads follow rivers; no road layer.
- Drop-one results unstable (without elevation ± 0.104) → combinations not interpretable at n = 76; report single-feature results.
- Aspect again looks like noise (without aspect > all terrain), but kept in the main model to avoid tuning on the same data.

### Figure 4 (Cell 9)
- Changed plan: no ROC curves (pooled ROC would show 0.62 while headline is within-block 0.72 → confusing).
- (a) Pooled spatial-CV AUC, naive vs matched sampling, for terrain / coordinates-only / circular. (b) Matched set, within-block AUC: all terrain + each feature alone. Error bars = std over 10 repeats. Dashed line = 0.5.
- Polish: "coin flip" moved into legends (text overlapped a bar); y-ticks limited to 0–1.0; panel (a) value labels moved above error bars (0.49 overlapped its error bar).
- Saved figures/fig4_model_evaluation.png (300 dpi) + .pdf.

### Publishing Step 3
- Published matched points as data/training_points_matched_wgs84.geojson (EPSG:4326, Cell 10); Drive copy training_points_matched.geojson stays in UTM.
- Saved 03_model.ipynb to GitHub; uploaded figures/fig4_model_evaluation.png.
- README: added Figure 4 section, updated Approach step 4, roadmap, repo structure, Run it step 5.
