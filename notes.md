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
NOTE: several single-draw results in this section were later revised — see "Robustness checks and averaged map" below for the final numbers.

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
- The std values only reflect block layout; they don't capture uncertainty from the small sample (76 points).

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
- Results (mean ± std over 10 repeats, ONE draw of stable points):
  - coordinates only: random 0.400 ± 0.040; spatial pooled 0.485 ± 0.028; within-block 0.386 ± 0.026
  - full (circular): random 0.942 ± 0.007; spatial pooled 0.928 ± 0.005; within-block 0.935 ± 0.004
  - terrain: random 0.669 ± 0.033; spatial pooled 0.623 ± 0.023; within-block 0.721 ± 0.048
- Circular model ≈ 0.93–0.94 even with matched points → bare ground marks the scar; NDVI/land cover excluded as predictors.
- (Superseded: single-draw headline 0.72 and "coordinates-only ≤ 0.5" → see across-draws results.)

### Feature diagnostics on matched points (Cell 8)
- Within-block AUC, 10 repeats, ONE draw: only slope 0.660 ± 0.065; only dist_river 0.669 ± 0.041; only elevation 0.563 ± 0.021; only aspect 0.422 ± 0.006.
- Without aspect 0.737 ± 0.050; without dist_river 0.716 ± 0.016; without elevation 0.520 ± 0.104; without slope 0.704 ± 0.044 (all terrain 0.721).
- Drop-one results unstable (without elevation ± 0.104) → combinations not interpretable at n = 76.
- (Superseded: slope 0.660 and aspect 0.422 were single-draw values → see across-draws results.)

### Figure 4 v1 (Cell 9) and publishing Step 3
- v1: (a) pooled spatial-CV AUC, naive vs matched; (b) single-draw within-block AUC per feature. Polished (legend, y-ticks, labels above error bars). Replaced by v2 (Cell 17).
- Published matched points as data/training_points_matched_wgs84.geojson (EPSG:4326, Cell 10); Drive copy stays in UTM.

## Figure 5: Susceptibility map (notebook 03_model) — first version (single draw)
- Cell 11: single RF (300 trees) on all 76 matched points; scored 1,767,254 pixels (≈ 1,590 km²); mean 0.380, median 0.348, IQR 0.215–0.525, range 0.033–0.968. High scores follow the river network; valleys also hold roads and villages.
- Cell 12: quintile classes (breaks 0.191 / 0.287 / 0.409 / 0.573); extrapolation hatching (> 5 km from any training point: 38.3% of modelled area, mostly south); training-data check 36 very high + 2 high (not validation).
- Cell 13 held-out check (single draw, uneven blocks 29/29/12/2/4): landslides 87% High/Very high vs stable 66%.
- Superseded by the averaged map (Cell 18), averaged held-out check (Cell 19) and Figure 5 re-run.
- README: disclaimer (student project, not an official hazard map; refer to USDMA). Repo fixes: restored Acknowledgements (went missing several times when pasting full files), re-added requirements.txt, added About description + topics.

## Step 4: Rainfall (notebook 04_rainfall)

### Scope (decided before analysis)
- Inventory has no failure dates → can't link landslides to specific storms or derive rainfall thresholds.
- IMERG pixels ~11 km; landslides and matched stable points (≤ 3 km apart) usually share a pixel → rainfall can't discriminate them; adding it to the model would reintroduce location. Rainfall NOT added to the model.
- Instead: (1) seasonality and year-to-year monsoon variability, (2) sanity check: June 2013 (Kedarnath disaster) should stand out, (3) spatial pattern of monsoon rain vs susceptibility.

### Monthly rainfall (Cells 1–2)
- Dataset: NASA/GPM_L3/IMERG_MONTHLY_V07, band "precipitation" (mm/hr, monthly mean rate). Monthly total = rate × days in month × 24.
- District mean via reduceRegion (mean, scale 5000 m) over the GAUL 2025 Rudraprayag boundary.
- Saved rainfall_monthly_rudraprayag.csv. Available period: 1998-01 to 2025-09 (333 months); full 2025 monsoon included. Early years (1998–2000) less reliable.
- Mean monthly totals (mm): Jan 77, Feb 91, Mar 92, Apr 75, May 95, Jun 176, Jul 477, Aug 431, Sep 253, Oct 73, Nov 19, Dec 22. Winter (Jan–Mar) not dry: western disturbances.
- Sanity check PASSED: June 2013 = wettest June on record (501 mm, ~2.8× the average June of 176 mm). Other wet Junes: 2011 (360), 2008 (358), 2000 (343), 2025 (287).
- Caveat: satellite rainfall in steep Himalayan terrain is uncertain; the June 2013 check is reassuring but not a gauge validation.

### Monsoon totals by year (Cell 3)
- 28 complete monsoons (1998–2025). Average 1,336 mm = 72% of average annual total (1,865 mm). Saved rainfall_monsoon_totals.csv.
- Wettest: 2010 1,924 mm (+44%), 2011 1,788 (+34%), 2013 1,773 (+33%), 2025 1,658 (+24%), 2018 1,606 (+20%). Driest: 2009 813 (−39%; consistent with the 2009 all-India drought), 2002 950 (−29%), 2014 1,018 (−24%).
- 2013 only the 3rd wettest monsoon despite the Kedarnath disaster → seasonal totals don't capture short, intense bursts.
- Recent: 2023 +5%, 2024 +17%, 2025 +24% → scars mapped after three above-average monsoons; "consistent with", not causal. No trend claim.

### Monsoon rainfall map (Cell 4)
- Mean June–September total per pixel, 1998–2025, clipped to district; downloaded at 1 km (real resolution ~11 km). Saved rudraprayag_monsoon_rain.tif.
- Check: district average from map 1,328 mm vs 1,336 mm. Bug: outside-district pixels came back as 0 → treated ≤ 0 as no data (Cell 5).
- Pattern: wettest NW (~1,600+ mm), wet middle band towards the east, driest SW (~940–1,100). Observation only (confounded with inventory location; 11 km pixels).

### Rainfall vs susceptibility (Cell 5)
- First run (single-draw map): class means 1,372 / 1,353 / 1,325 / 1,316 / 1,320 mm; "Very high" in wettest third 33%.
- Re-run on the averaged map: class means (very low → very high) 1,368 / 1,341 / 1,329 / 1,326 / 1,322 mm (~3.5% spread); "Very high" in wettest third 33% (unchanged) → susceptibility and monsoon rainfall independent at this scale.
- Points: landslides 1,324 mm vs stable 1,310 mm; 26 of 38 landslides share exactly the same rain value as their nearest stable point → rainfall can't discriminate at this resolution.

### Figure 6 (Cell 6) and publishing Step 4
- (a) mean monthly rain with 10th–90th percentile range, June 2013 marked; (b) monsoon totals 1998–2025, 2013 and 2025 highlighted and annotated; (c) average monsoon rain map with landslides. Polish: wider panel (c); legend in (b) moved to top centre.
- Published 04_rainfall.ipynb (re-saved after the averaged-map re-run), figures/fig6_rainfall.png, data/rainfall_monthly_rudraprayag.csv, data/rainfall_monsoon_totals.csv. IMERG citation: Huffman et al. (2019), doi:10.5067/GPM/IMERG/3B-MONTH/07. Figure 6 unaffected by the averaged map.

## Step 5: Interactive map (notebook 05_interactive_map)
- Folium (Leaflet) map published with GitHub Pages (docs/index.html): https://makkergauri.github.io/uttarakhand-landslide-risk/
- Web layers (Cells 1–2): classes reprojected UTM 44N → Web Mercator (EPSG:3857, nearest neighbour), coloured RGBA PNGs (susceptibility + extrapolation stripes), lat/lon bounds; embedded as base64 → one self-contained HTML file (~1.1 MB).
- Map (Cell 3): basemaps Esri World Imagery (default), OpenStreetMap, OpenTopoMap (show=False on the extra ones, otherwise all display at once); susceptibility (opacity 0.6), stripes, district boundary (white line, dark casing), 38 landslide markers with popups (coordinates, class, Google Maps satellite link, "not field-verified"); legend bottom left with disclaimer (auto-folds on screens < 700 px); title; scale bar.
- Colab preview crowded/zoomed out (small iframe) → judged on the real site. Faint lines across basemap = tile gaps at browser zoom ≠ 100%, not data.
- Downloads: browser renames repeated downloads to "name (1).ext" → always rename to exactly index.html before uploading.
- Rebuilt from the averaged map: done; docs/index.html replaced. Check: popups of several landslides all show "Very high" (as in Cell 12's 38/38), confirming the live site uses the averaged map.
- Tests: street map loads outside Colab ✔; Google Maps link opens satellite view at the scar ✔; phone: legend starts folded and opens on tap ✔, title readable, no overlap with layer box ✔.

## Figure 1: Study area (notebook 06_study_area)
- Boundary sensitivity: GAUL draws disputed areas separately, so a GAUL "India" outline differs from the official Indian map → panel (a) shows all land in plain grey with no borders drawn; source note says boundaries from FAO GAUL 2025, not authoritative. For an Indian journal, Survey of India boundaries may be required.
- GAUL 2025 has no level0 layer → used FAO/GAUL/2025/level1 (169 units in lon 66–99, lat 5–38), simplified 5 km, clipped, drawn with edge colour = fill.
- Districts: 13 (GAUL1_NAME contains "Uttarakh", not "Uttar", which would match Uttar Pradesh). Main rivers: HydroSHEDS FFR, DIS_AV_CMS > 20 m³/s → 26 segments, 30–312 m³/s.
- Places (to verify in Google Maps): Rudraprayag 30.2847, 78.9812; Gaurikund 30.6533, 79.0250; Kedarnath 30.7352, 79.0669 — all inside the district. Rudraprayag dot sits exactly at the Mandakini–Alaknanda confluence from HydroSHEDS (independent check). Google Maps check: all three pins land on the right place (Kedarnath temple, Gaurikund village, Rudraprayag town at the confluence) ✔.
- Figure: (a) grey land, Uttarakhand red; (b) 13 districts, Rudraprayag label moved outside with arrow; °E/°N ticks; (c) hillshade coloured by elevation (700–7,000 m), rivers (width ∝ flow), places, scale bar, north arrow, colourbar; source note.
- study_area/ GeoJSONs NOT published (FAO GAUL licence restricts redistribution); notebook regenerates them.

## Robustness checks and averaged map (notebook 03_model, Cells 14–19)

### Slope circularity check (Cell 14)
- Candidate filter required slope > 25° → re-drew stable points with the same rule (plus same matching). Saved training_points_matched_steep.geojson.
- Single draw: landslide centres ≤ 25°: 2 of 38; slope means now equal (37.2 vs 37.5°). Within-block AUC (slope-matched vs original): terrain 0.536 vs 0.721; only slope 0.637 vs 0.660; only dist_river 0.550 vs 0.669; coordinates 0.475 vs 0.386.
- Key realisation: all ± values so far only reflected block layout, not WHICH stable points were drawn.

### Across-draws uncertainty (Cells 15–16)
- Slope distributions (p10 / median / p90 / share > 45°): original landslide 25.8 / 36.9 / 48.0 / 0.18 vs stable 19.8 / 36.7 / 45.5 / 0.13; slope-matched stable 27.0 / 36.5 / 47.4 / 0.26 → original slope "signal" was the gentle tail = the candidate filter.
- 20 re-draws per design (seeds 100–119, new block layout per draw). Within-block AUC, mean (95% range), original matched / slope-matched:
  - terrain: 0.744 (0.515–0.850) / 0.694 (0.467–0.862); > 0.6 in 95% / 80% of draws
  - only dist_river: 0.643 (0.448–0.849) / 0.629 (0.439–0.794)
  - only aspect: 0.635 (0.465–0.780) / 0.594 (0.370–0.778)
  - only elevation: 0.544 (0.333–0.726) / 0.570 (0.377–0.727)
  - only slope: 0.542 (0.396–0.737) / 0.482 (0.320–0.694)
  - coordinates only: 0.585 (0.485–0.685) / 0.532 (0.392–0.637)
- Terrain beats coordinates in 90% of draws in both designs (average gap +0.16) → most robust finding.
- Decision (rules set in advance): ranges overlap → slope rule doesn't clearly change the overall result (mean −0.05).
- Superseded single-draw claims: "AUC 0.72" → ~0.74 (0.52–0.85); "slope and dist_river ~0.66" → slope ≈ chance, dist_river ~0.64; "aspect adds nothing" → ~0.6; "coordinates ≤ 0.5" → ~0.59.
- Methods lesson: with small inventories, report across many background draws, never one. Saved data/auc_across_draws.csv.

### Figure 4 v2 (Cell 17)
- (a) unchanged, title "(one draw)"; (b) box plots across 20 draws for both designs (boxes = middle 50%, whiskers = 95%, line = median); caption must note (b) y-axis starts at 0.2.

### Averaged map (Cell 18)
- 20 RFs (100 trees each, min_samples_leaf = 2, random_state = draw), one per draw (original matched design, seeds 100–119), each scoring the whole domain. Mean → rudraprayag_susceptibility.tif; sd across models → rudraprayag_susceptibility_sd.tif; single-draw map → rudraprayag_susceptibility_single_draw.tif.
- Single-draw vs averaged: r = 0.90; 60% of pixels same class, 95% within one class → pattern robust, classes = broad bands.
- Mean disagreement (sd) 0.088 (max ~0.2); highest ALONG THE RIVERS (my prediction "highest in the south" was wrong): models agree valleys are riskier but not how much.
- Averaged score: mean 0.364, median 0.329, IQR 0.217–0.490, range 0.046–0.961.
- Bug: after a Colab reconnect, re-running Cell 11 overwrote the averaged map (same file name) → Cell 11 now saves to _single_draw.tif; re-ran Cell 18. All outputs identical to the first run → fixed seeds make the pipeline reproducible.

### Averaged held-out check (Cell 19)
- Blocks: k-means (k = 5, random_state 42) on the 38 landslides only → 6, 6, 14, 2, 10 landslides (old blocks very uneven). Each draw's stable points assigned to the nearest block.
- For each block: 20 models trained without it; their average scores held-out landslides, held-out stable points from all 20 draws, and a 100,000-pixel sample (for the fold's quintile breaks).
- Held-out classes (landslides / stable, %): very low 0/10; low 5/22; moderate 8/19; high 26/26; very high 61/23.
- Landslides in High/Very high: 87% (33/38; 95% range 73–94%) vs stable 49% (760 points pooled over 20 draws; same areas reused → more precise than landslides but not 760 independent points). Chance 40%. Very high alone: 61% vs 23%.
- Single-draw stable 66% was on the high side; gap now 38 points → the map separates failing slopes from neighbours, not just valleys.

### Figure 5 re-run from the averaged map (Cell 12)
- Class breaks: 0.193 / 0.285 / 0.383 / 0.538 (old single-draw: 0.191 / 0.287 / 0.409 / 0.573). Extrapolation 38.3%. Training-data check: 38/38 very high (all 20 models saw all landslides → in-sample, not validation).
- Correction: the continuous averaged scores are calmer in the south, but the CLASS map still looks mixed there, because southern scores sit near the class breaks (~0.3–0.45) → south = extrapolation, don't read closely.
- Scale bar moved to lower right (the earlier fix had never been applied). Upload mix-up: browser saved re-downloads as "name (1).png" → README showed old figures; fixed by renaming locally and re-uploading under the original names (GitHub can't rename binary files in the browser).
- Figure 5 re-run from the averaged map; README updated (Figure 5 section rewritten: averaged map, held-out 87% vs 49%, 61% vs 23%, classes as broad bands, disagreement along rivers, south caveat; Approach step 4; roadmap; repo structure; Run it reproducibility note).
