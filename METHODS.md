# Methods

Perennial snow/ice mapping over the Beas Kund cirque (Himachal Pradesh, Indian Himalaya) from fused
Sentinel-2 and Landsat 8/9 optical imagery, 2017–2024, with independent cross-validation over the
Mont Blanc massif against the operational Theia LIS snow-cover-duration product.

---

## 1. Study area & grid

The primary AOI is the Beas Kund cirque, bounding box `[77.03373, 32.31549, 77.14879, 32.41271]`
(WGS84), projected to UTM 43N (EPSG:32643). All imagery is resampled to a common **30 m** grid — the
native Landsat resolution and the DEM resolution — with Sentinel-2 **downsampled** by area-averaging
(never Landsat upsampled), so no spatial detail is fabricated.

## 2. Multi-sensor acquisition model

Three sensors are fused: Sentinel-2 L2A, Landsat 8 C2L2, and Landsat 9 C2L2 (Microsoft Planetary
Computer STAC). Each `(tile, date, sensor)` acquisition is treated as an **independent unit** rather
than compositing per date. Because Landsat's orbit yields clear-sky days independent of Sentinel-2's,
the fusion fills cloud gaps in the ablation-minimum window that single-sensor coverage leaves — coverage
was particularly thin in 2022 and 2024 under Sentinel-2 alone.

Sensor front-ends differ and converge after producing surface reflectance and boolean masks:

| | Sentinel-2 | Landsat 8/9 |
|---|---|---|
| green / SWIR1 | B03 / B11 | SR_B3 / SR_B6 (`green`/`swir16`) |
| reflectance | (DN − 1000) / 10000 | DN × 0.0000275 − 0.2 |
| cloud/water mask | SCL integer classes | QA_PIXEL bit flags (CFMask) |

## 3. Per-look classification

For each acquisition, the Normalized Difference Snow Index is computed,
`NDSI = (green − SWIR1) / (green + SWIR1)`, with **reflectance screens applied before the index**
following the MODIS/VIIRS snow-cover ATBD:

- reflectance clipped to `[0, 1]` (removes Sen2Cor over-unity artifacts on bright snow and keeps
  genuinely bright pixels);
- both bands required `> 0` and green `> 0.10` (low-VIS screen), which removes unmasked nodata
  (Sentinel DN 0 → −0.1 reflectance) and rejects dark-surface false snow.

Without these screens NDSI took values far outside `[-1, 1]` (observed −6446 to +2552) from nodata and
over-unity reflectance entering the ratio. A pixel is **snow** where `NDSI > 0.40` (the standard
MODIS/Sentinel threshold).

Cloud masking is sensor-specific (SCL classes; QA_PIXEL cloud/cirrus/dilated bits). **NDSI overrides
the cloud flag on snow pixels**: both SCL and Landsat CFMask systematically mislabel bright snow as
cloud, so a pixel reading NDSI-snow is retained even if flagged cloudy. Cloud shadow, fill, and water
are hard-masked; water is retained as a separate known-surface state, not treated as no-data.

Each look is reduced to one of four states: snow, not-snow, masked (abstain), water.

## 4. Per-acquisition cloud gating

Before classification, each acquisition's obstruction fraction is measured over **its own footprint**
(not the whole AOI). Acquisitions with obstruction above 0.60 are dropped. Per-footprint gating means a
cloudy Sentinel tile does not disqualify a clear Landsat pass on the same date.

## 5. Per-year verdict

Each year's stack of clear looks is reduced to one verdict per pixel. **Bare status is a yearly
aggregate, never a per-epoch veto**: a pixel is perennial-that-year if snow appears in ≥ 80% of that
year's clear *land* looks (water excluded), bare-that-year if ≤ 20%. A year is *decided* only with ≥ 2
clear land looks; thinner years abstain. This tolerates the noise minority (thin cloud, off-nadir
geometry, mixed edge pixels) that a per-epoch bare veto would let disqualify a genuinely perennial pixel.

**Nadir-recency confidence.** Each snow look carries a weight peaking on the ablation-minimum plateau
and decaying for too-early (incomplete melt) or too-late (fresh-snow) last-looks. A pixel's per-year
confidence is its best (highest-weighted) snow look. This is a phenology-grounded heuristic — the tent
shape and breakpoints are an engineering choice informed by melt/accumulation timing, not a derived
formula — introduced because this AOI's ablation window is cloud-scarce and off-nadir looks must be
admitted but down-weighted rather than trusted equally.

## 6. Ablation-minimum window

The nadir plateau is grounded per-AOI, because the ablation minimum is climate-regime specific:

- **Beas Kund (monsoon-facing):** late August – mid September, from (a) this project's own pooled
  snow-fraction-vs-day-of-year curve (minimum ~30 Aug – 1 Sept) and (b) the same-basin Beas River Basin
  MODIS/ERA5 study (August SCA minimum). An arid trans-Himalayan reference (e.g. Zanskar) was
  deliberately *not* used — different precipitation regime.
- **Mont Blanc (validation AOI):** mid-to-late September, from Alpine glaciology literature. The Alpine
  minimum is *later* than the Himalayan one; reusing the Himalayan window in the Alps would test snow
  presence while seasonal snow is still melting and over-claim perennial.

## 7. Multi-year persistence & classification

Per-year verdicts are combined into a single class per pixel. Multi-year confidence is the mean of the
(nadir-weighted) yearly confidences over decided years.

**Asymmetric evidence:**
- **PERENNIAL** requires `≥ MIN_YEARS` (3) decided years, mean confidence `≥ PERENNIAL_W` (0.75), and
  bare in `≤ MAX_BARE_YEARS` (2) years.
- **NOT-PERENNIAL** fires on bare in `> MAX_BARE_YEARS` years regardless of decided-year count — ruling
  perennial *out* needs only positive bare evidence, whereas asserting it needs multi-year persistence.

**Explicit non-verdict labels** — every pixel carries a reason: NO_DATA (masks rejected every look),
WATER (known surface, excluded), UNDECIDED (observed but too thin), plus a deferred CANDIDATE class
(undecided pixels fully ringed by perennial).

**The confidence gate is the dominant filter and behaves as a data-quality signature.** At Beas Kund it
rejects ~74% of pixels that pass the persistence structure, because under sparse monsoon-influenced
coverage many persistent pixels are only ever seen snow off-nadir and are correctly withheld rather than
over-claimed. Perennial extent is therefore threshold-sensitive under scarcity (varies 4.6× across
`PERENNIAL_W` ∈ [0.5, 0.8]) but far less so under dense coverage (1.3× at Mont Blanc), confirming the
gate is near-inert where data is rich and discriminating where sparse.

## 8. Interannual change

Change is measured on a **fixed observable intersection** — pixels with a binary perennial-that-year
verdict in *every* study year (2019, 2022, 2023, 2024) — so that a change in the perennial count reflects
real change on the same ground rather than year-to-year sampling drift. The study years are a *subset* of
the mask years (all 8): the full 8-year intersection is too small (~2.2k px) because thin years
(2017/18/20/21) require a verdict everywhere, whereas dropping them grows the comparable set to ~33k px.
The verdict unit is strictly binary; confidence-value deltas are not differenced across years (sensor and
timing differences make confidence *values* non-comparable). The trend is reported on the reliably-observed
subset (~25% of the AOI), not basin-wide, and no aspect-differentiated melt claim is made.

## 9. Validation

The method is cross-validated over the Mont Blanc massif (Theia tile T32TLR, UTM 32N) against the
**Theia Sentinel-2 Snow Cover Yearly Synthesis (LIS Level-3B)** snow-cover-duration (SCD) product,
hydrological years 2019–2024. No operational snow product covers the Himalayan AOI (Copernicus HR-S&I
was decommissioned in 2025), so the AOI-agnostic pipeline is validated where a reference exists.

Theia SCD (days of snow cover per hydrological year) is thresholded and aggregated to a multi-year
perennial mask matching the method's own tolerance, reprojected to the pipeline grid, and compared over
pixels where both products have a verdict. Agreement is reported as Cohen's kappa, precision, recall, F1,
and the full confusion matrix, swept across SCD thresholds for robustness.

**How Theia LIS builds the reference.** LIS produces a per-date snow map from NDSI (Dozier 1989) plus a
red-band test, run in two passes with a **DEM-derived snowline** that excludes snow below a per-scene
minimum elevation zs (Gascoin et al. 2019). The Level-3B **yearly synthesis** (Barrou Dumont et al. 2025) then aggregates these per-date maps over a
hydrological year (1 Sep–31 Aug) into **SCD — snow-cover duration, the total number of snow-covered days
in the year** (gap-filled from the ~5-day Sentinel-2 revisit). SCD is a duration count (0–~365), not a
native perennial classification.

**Building a perennial proxy from SCD.** A permanent-snow/ice pixel is snow-covered essentially every
day, so its SCD approaches 365; seasonal snow scores far lower. Perennial is therefore defined here as
**SCD ≥ 360 days** (snow essentially the entire hydrological year) held across study years — a definitional
threshold, not one tuned to maximise agreement. Agreement was checked across the full SCD range (see
below) and is high throughout; 360 is chosen because it *defines* perennial, independent of fit. This is
explicitly a *proxy*: SCD measures duration, and this pipeline is the only one of the two that natively
classifies perennial snow — the comparison converts Theia's duration into a perennial mask to compare
like with like.

**Result (at the definitional threshold SCD ≥ 360):** Cohen's kappa 0.86, precision 0.94 — the method does
not over-detect perennial snow relative to the reference. Agreement is high and stable across the whole SCD
range (kappa 0.83–0.87 for 330–360 d), so the result does not depend on the threshold choice; 360 is used
because it is the definitional perennial cut, not a fitted one.

**Every disagreement pixel is explained — none is classification error.** The comparison is over decided
pixels only (both products returned a verdict), so disagreements are genuine class differences, not
masked-out cells.

*Ours-only* (the method calls perennial, Theia does not; ~11,100 px, low elevation, median ~2700 m): these
trace **glacier tongues** (Mer de Glace, Argentière), visible directly on the disagreement map as coherent
sinuous downslope features. Both products detect snow/ice by NDSI, so the difference is **not** a detection
difference — it is the one documented algorithmic difference between them: Theia's LIS applies a **DEM
snowline that excludes snow below a per-scene minimum elevation** zs (Gascoin et al. 2019), and glacier
tongues descend below that snowline. The method has no snowline gate, so it retains them, mapping the full
snow-and-ice cryosphere extent rather than snow above a snowline.

*Theia-only* (Theia calls perennial, the method does not; ~4,600 px, high elevation, median ~3220 m) is a
limitation on the *method's* side, not evidence that Theia over-includes. It splits into ~88%
**low-confidence** (snow observed across enough years but below the nadir-weighted confidence bar —
conservative under off-nadir or thin evidence) and ~12% **shadow-affected**. The shadow group was checked
by aspect: these pixels sit on markedly more north-facing (northness +0.62 vs +0.25 for agreed-perennial)
and steeper (33° vs 25°) terrain than agreed-perennial pixels — a clear topographic-shadow correlation. They
are also marginal (89% are bare in exactly three of six years, one past the two-year tolerance). The
correlation with shaded, steep, north-facing terrain indicates these are shadow-affected pixels where the
snow verdict is unreliable; we do not claim a specific radiometric mechanism (whether shadow intermittently
depresses NDSI below threshold, trips the shadow mask, or both is not established here). What the data
supports is the association: the method's few disagreements with Theia at high elevation fall on exactly the
shadowed terrain where optical snow mapping is hardest, so the method slightly under-detects perennial snow
on north-facing steep slopes — the same aspect-linked observational limit noted for the change study.
Neither Theia-only group is classification error.

---

## References

- **Selkowitz, D. J. & Forster, R. R.** — automated mapping of persistent ice and snow cover from
  Landsat (USGS); late-summer NDSI persistence, validated against the Randolph Glacier Inventory.
  *(Method precedent — this pipeline was designed independently and later found to converge with it.)*
  *(Complete the exact citation: Selkowitz & Forster, Remote Sensing, 2016; and/or Selkowitz et al. 2017.)*
- **Gascoin, S., Grizonnet, M., Bouchet, M., Salgues, G. & Hagolle, O. (2019).** Theia Snow collection:
  high-resolution operational snow cover maps from Sentinel-2 and Landsat-8 data. *Earth System Science
  Data* 11(2):493–514. doi:10.5194/essd-11-493-2019. Product DOI: doi:10.24400/329360/F7Q52MNK.
- **Dozier, J. (1989).** Spectral signature of alpine snow cover from the Landsat Thematic Mapper.
  *Remote Sensing of Environment* 28:9–22. *(Original NDSI, which both this method and LIS build on.)*
- **Let-It-Snow (LIS) ATBD** — Algorithm Theoretical Basis Documentation for the operational snow
  product from Sentinel-2 and Landsat-8. Zenodo: zenodo.org/records/1414452.
- **Barrou Dumont, Z., Gascoin, S., Inglada, J., Dietz, A., Köhler, J., Lafaysse, M., Monteiro, D.,
  Carmagnola, C., Bayle, A., Dedieu, J.-P., Hagolle, O., and Choler, P. (2025).** Trends in the annual
  snow melt-out day over the French Alps and Pyrenees from 38 years of high-resolution satellite data
  (1986–2023). *The Cryosphere* 19(7):2407–2429. doi:10.5194/tc-19-2407-2025.
  *(Methodology for the Theia Level-3B annual synthesis — SMOD/SCD/SOD maps — which are the validation
  reference. The L3B products are built from the per-date LIS snow maps of Gascoin et al. 2019.)*
- **Beas River Basin MODIS/ERA5 study** — same-basin snow-cover-area seasonality (August SCA minimum).
  *(Ablation-window grounding for Beas Kund.)*
- **Kavan et al. 2021; Paul et al. 2020; Rastner et al. 2019** — Alpine ablation-minimum timing
  (mid-to-late September). *(Ablation-window grounding for the Mont Blanc validation AOI.)*
- **MODIS / VIIRS snow-cover ATBD** — reflectance and low-VIS screens applied before NDSI.
- Copernicus **GLO-30 DEM** — elevation for the validation disagreement analysis.

> Full citation details (authors, venues, DOIs) to be completed before publication — several entries
> above are identified by description and need their bibliographic records filled in.
