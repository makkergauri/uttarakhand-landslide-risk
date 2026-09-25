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

## Figure 5: Susceptibility map (notebook 03_model)

### Prediction (Cell 11)
- Final model: same RF settings (300 trees, min_samples_leaf = 2, random_state 42), trained on all 76 matched points, terrain features only.
- Prediction domain = same as training: elevation 0–3500 m, not WorldCover 70/80. Outside domain = NaN ("not modelled").
- Scores are relative, not true probabilities (training was 1:1 landslide:stable, not real landslide frequency) → map uses classes, not raw scores.
- Saved rudraprayag_susceptibility.tif (float32, EPSG:32644, 30 m, NaN = not modelled).
- Pixels scored: 1,767,254 (≈ 1,590 km² at 30 m). Score summary: mean 0.380, median 0.348, IQR 0.215–0.525, range 0.033–0.968.
- Quick look: high scores follow the river network (dist_river signal), with valley walls higher than ridges (slope signal). Physically plausible (valley-side slides along Mandakini/Alaknanda and tributaries), but valleys also hold roads and villages → can't separate "near river" from "near road".
- Northern peaks > 3,500 m and snow/ice not modelled; the model reaches only up the valleys.

### Figure 5 (Cell 12)
- Classes: quintiles of the modelled area (each class = 20% of area): very low / low / moderate / high / very high. Chosen because scores are relative, not probabilities.
- Extrapolation flag: hatching where a modelled pixel is > 5 km from any training point (landslide or matched stable point).
- Landslide class counts reported as a training-data consistency check only (model has seen these points), NOT validation.
- Layout: grey = not modelled inside district, district outline, white triangles = 38 landslides, scale bar, north arrow, legend.
- Class breaks (score): 0.191 / 0.287 / 0.409 / 0.573.
- Modelled area > 5 km from any training point: 38.3% (mostly south) → over a third of the map is extrapolation.
- Landslides by class (training data): very high 36, high 2, others 0; none outside the modelled area. Expected: a random forest nearly memorises 76 training points → NOT evidence the map is right. Honest performance stays within-block AUC ≈ 0.72.
- South (hatched) shows speckled red/orange/yellow instead of the clean river-line pattern: south is lower (700–1,500 m) than all training data, and landslides sit slightly lower than their neighbours → model extrapolates "lower = riskier". Likely artefact, flagged by hatching.
- Polish: legend moved outside the map on the right (district fills every corner); scale bar moved to lower right (covered the SW tip of the district).
- Saved figures/fig5_susceptibility_map.png (300 dpi) + .pdf.

### Held-out class check (Cell 13)
- Method: k-means blocks (k = 5, random_state 42) on matched points. For each block: train RF on the other 4 blocks, score the whole district, classify with that model's own quintiles, record the class of each held-out point. Every point scored by a model that never saw it.
- Baseline: classes are 20% of area each → 40% of landslides in High/Very high by chance.
- Compare held-out landslides vs held-out stable points; the gap = how well the map separates failing slopes from nearby stable ones.
- Block sizes very uneven: 29, 29, 12, 2, 4 held-out points (two northern clusters hold most points) → holding out a big block leaves only 47 training points.
- Held-out classes (stable / landslide): very low 2/0; low 5/2; moderate 6/3; high 14/17; very high 11/16.
- Held-out landslides in High/Very high: 87% (33/38), approx. 95% range 73–94%. Chance level 40%.
- Held-out stable points in High/Very high: 66% (25/38), approx. 95% range 50–79%.
- Interpretation: map reliably places unseen landslides in top classes, but nearby stable slopes also score high (they were sampled in the same valleys) → the map mainly identifies hazardous valley sides; slope-level separation is partial (consistent with within-block AUC 0.72). Ranges overlap → gap suggestive, not conclusive. Single block layout (seed 42) only.

### Publishing Figure 5
- Saved 03_model.ipynb to GitHub again (Cells 11–13); uploaded figures/fig5_susceptibility_map.png.
- README: added Figure 5 section, disclaimer (student project, not an official hazard map; refer to USDMA), roadmap, requirements.txt in repo structure.
- Repo fixes after check: restored Acknowledgements in README (went missing several times when pasting full files), re-added requirements.txt (had gone missing), added About description + topics.

## Step 4: Rainfall (notebook 04_rainfall)

### Scope (decided before analysis)
- Inventory has no failure dates → can't link landslides to specific storms or derive rainfall thresholds.
- IMERG pixels ~11 km; landslides and matched stable points (≤ 3 km apart) usually share a pixel → rainfall can't discriminate them; adding it to the model would reintroduce location. Rainfall NOT added to the model.
- Instead: (1) seasonality and year-to-year monsoon variability, (2) sanity check: June 2013 (Kedarnath disaster) should stand out, (3) spatial pattern of monsoon rain vs susceptibility.

### Monthly rainfall (Cells 1–2)
- Dataset: NASA/GPM_L3/IMERG_MONTHLY_V07, band "precipitation" (mm/hr, monthly mean rate). Monthly total = rate × days in month × 24.
- District mean via reduceRegion (mean, scale 5000 m) over the GAUL 2025 Rudraprayag boundary.
- Saved rainfall_monthly_rudraprayag.csv (month, rate_mm_hr, mm, year, mon).
- Available period: 1998-01 to 2025-09 (333 months). Full 2025 monsoon (June–September) included.
- Early years (1998–2000) use fewer satellites → less reliable.
- Mean monthly totals (mm): Jan 77, Feb 91, Mar 92, Apr 75, May 95, Jun 176, Jul 477, Aug 431, Sep 253, Oct 73, Nov 19, Dec 22. June–September ≈ 1,337 mm ≈ 71% of annual (~1,880 mm). Winter (Jan–Mar) not dry: western disturbances.
- Sanity check PASSED: June 2013 = wettest June on record (501 mm, ~2.8× the average June of 176 mm); Kedarnath disaster rain fell mostly within a few days, so even this monthly total understates the intensity.
- Other wet Junes: 2011 (360), 2008 (358), 2000 (343), 2025 (287, 5th wettest).
- Caveat: satellite rainfall in steep Himalayan terrain is uncertain (localised orographic rain); the June 2013 check is reassuring but not a gauge validation.

### Monsoon totals by year (Cell 3)
- June–September sums for years with all 4 months; % difference from the average; saved rainfall_monsoon_totals.csv.
- 28 complete monsoons (1998–2025). Average 1,336 mm = 72% of average annual total (1,865 mm).
- Wettest: 2010 1,924 mm (+44%), 2011 1,788 (+34%), 2013 1,773 (+33%), 2025 1,658 (+24%), 2018 1,606 (+20%).
- Driest: 2009 813 mm (−39%; consistent with the 2009 all-India drought), 2002 950 (−29%), 2014 1,018 (−24%).
- 2013 is only the 3rd wettest monsoon despite the Kedarnath disaster → seasonal totals don't capture the short, intense bursts that trigger landslides (June 2013 = most extreme June). Supports day-scale rainfall for early warning, not seasonal totals.
- Recent monsoons: 2023 1,397 (+5%), 2024 1,559 (+17%), 2025 1,658 (+24%, 4th wettest). Scars mapped Oct–Dec 2025 followed three above-average monsoons → "consistent with", not causal (inventory has no dates).
- Do NOT claim a trend: 28 noisy years, early years less reliable.

### Monsoon rainfall map (Cell 4)
- Mean June–September total per pixel, 1998–2025 (monthly rate × hours in month, summed per year, averaged over years), clipped to district.
- Downloaded via getDownloadURL at 1 km, EPSG:32644, outside = −9999 → NaN. Real resolution still ~11 km (≈ 30 pixels across the district).
- Saved rudraprayag_monsoon_rain.tif.
- Check passed: district average from map 1,328 mm vs 1,336 mm from monthly series (Cell 3).
- Bug: outside-district pixels downloaded as 0, not −9999 → first min showed 0 mm. Fixed in Cell 5 by treating ≤ 0 as no data (no real monsoon total is 0).
- Pattern: wettest in the northwest (~1,600+ mm, upper Mandakini side towards Kedarnath/Gaurikund), wet band across the middle towards the east (~1,400–1,500), driest in the southwest (~900–1,100). Roughly 2× range within one district.
- NW wettest AND many landslides in NW, but north–south differences in this dataset are confounded with inventory location, and pixels are 11 km → observation only, not an explanation of the clustering.

### Rainfall vs susceptibility (Cell 5)
- Rain map reprojected (nearest) onto the 30 m susceptibility grid; same quintile classes as Figure 5.
- Metrics: mean/median monsoon rain per class; share of "Very high" area in the wettest third (≈ 33% = no relationship); rain at landslides vs stable points; how many landslides share exactly the same rain value as their nearest stable point (= same IMERG pixel).
- Fixed range: 940–1,663 mm per monsoon.
- Mean monsoon rain by susceptibility class (mm): very low 1,372; low 1,353; moderate 1,325; high 1,316; very high 1,320 (~4% spread).
- "Very high" area in the wettest third: 33% = exactly the no-relationship value → terrain susceptibility and monsoon rainfall exposure are independent at this scale → the two maps carry separate information; overall hazard depends on both.
- Rain at points: landslides 1,324 mm vs stable 1,310 mm. 26 of 38 landslides have exactly the same rain value as their nearest stable point (same IMERG pixel); the rest in neighbouring pixels with similar values → confirms rainfall can't discriminate at this resolution; not added to the model (my prediction was 30+; 26 is slightly fewer, same conclusion).

### Figure 6 (Cell 6)
- (a) Mean monthly rain with 10th–90th percentile range across years; monsoon months dark blue; June 2013 as red point.
- (b) Monsoon totals 1998–2025; 2013 and 2025 highlighted; average line; annotations ("2013: Kedarnath disaster, only 3rd wettest season"; "2025: before my scars were mapped").
- (c) Average monsoon rain map (1 km display of ~11 km data), district outline from the 30 m feature stack, landslides, scale bar, colourbar.
- Polish: wider slot for panel (c) (map was too small; figsize 16×5.2, width ratios 1 / 1.5 / 1.25); panel (b) y-limit raised to 2,700 and average legend moved to upper centre (was covering bars).
- Saved figures/fig6_rainfall.png (300 dpi) + .pdf.

### Publishing Step 4
- Saved 04_rainfall.ipynb to GitHub; uploaded figures/fig6_rainfall.png.
- Published data/rainfall_monthly_rudraprayag.csv and data/rainfall_monsoon_totals.csv.
- README: Figure 6 section (monsoon share, June 2013 check, 2013 only 3rd wettest season, 2023–2025 above average, spatial pattern, rain–susceptibility independence, why rainfall not in model, satellite-rain caveat); IMERG citation (Huffman et al. 2019, doi:10.5067/GPM/IMERG/3B-MONTH/07); roadmap (Step 5 next, Figure 1 added); repo structure; Run it step 6.

## Step 5: Interactive map (notebook 05_interactive_map)

### Plan
- Folium (Leaflet) web map, published with GitHub Pages (docs/index.html).
- Layers: satellite basemap (Esri World Imagery) + street map (OpenStreetMap; blocked inside Colab but works in normal browsers) + topographic (OpenTopoMap); susceptibility classes (Figure 5 colours, toggleable, semi-transparent); extrapolation zone (> 5 km from training points) as stripes; 38 landslides with popups linking to Google Maps satellite for self-checking; district boundary; legend; disclaimer.

### Web layers (Cells 1–2)
- Classes rebuilt with the same quintile breaks as Figure 5; extrapolation zone recomputed (> 5 km from any matched training point).
- Reprojected UTM 44N → Web Mercator (EPSG:3857), nearest neighbour (classes must stay classes); web maps use EPSG:3857, so an unconverted image would sit slightly off.
- Coloured to RGBA PNGs (not modelled = transparent; stripes for extrapolation). Saved web/susceptibility_3857.png, web/extrapolation_3857.png, web/image_bounds.json (lat/lon corners).
- Checks passed: class breaks identical to Figure 5 (0.191 / 0.287 / 0.409 / 0.573); extrapolation area 38.3% (same as Figure 5).
- Web Mercator image: 1,773 × 2,366 px; bounds lat 30.1729–30.8119, lon 78.8076–79.3634. PNGs: susceptibility 0.7 MB, stripes < 0.05 MB.

### Map (Cell 3)
- Basemaps: Esri World Imagery (default), OpenStreetMap, OpenTopoMap (attributions included).
- Overlays: susceptibility PNG (opacity 0.6), extrapolation stripes, district boundary (GAUL 2025, white line with dark casing), 38 landslides as white circle markers with tooltip + popup (coordinates, class at point, Google Maps satellite link, "not field-verified" note).
- PNGs embedded as base64 data URLs → the map is one self-contained HTML file (web/index.html).
- Legend (collapsible via <details>) with classes, stripes, marker, "not modelled", disclaimer (student project, not official; refer to USDMA), link to repo. Title bar at top; scale bar bottom left; layer control expanded.
- index.html: 1.1 MB.
- Fixes after first render: all three basemaps were visible at once (Folium shows every TileLayer by default) → show=False on street and topo maps; legend covered the layer control → moved to bottom left (above scale bar), max-width 230 px; marker radius 6 → 5 (merged into a blob when zoomed out).
- Colab preview crowded and zoomed out (small iframe ~450 px tall) → judged on the real site instead.
- Legend auto-folds on screens < 700 px wide (small script; <details> removes "open").

### Publishing (Step 5d)
- Downloaded index.html via google.colab files.download (Drive search box doesn't understand folder paths).
- Saved 05_interactive_map.ipynb to GitHub.
- Map uploaded as docs/index.html; GitHub Pages enabled (Deploy from a branch → main → /docs).
- Site: https://makkergauri.github.io/uttarakhand-landslide-risk/
- Laptop test: opens zoomed to the district (Colab zoom issue was the small iframe); popups work (e.g. #19, Very high); legend and layer control don't overlap.
- Faint lines across the basemap = gaps between satellite tiles when browser zoom / display scaling ≠ 100% (known Leaflet quirk), not a data issue.
