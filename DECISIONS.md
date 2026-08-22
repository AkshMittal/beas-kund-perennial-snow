# Design Decisions

The reasoning behind the choices in this pipeline. Each entry states the
decision, why it was made, and — where it matters — the approaches that were tried first
and abandoned, because the rejected paths explain the final design as much as the choice does.

---

## 1. Multi-year persistence, not single-scene snow

Perennial snow is defined as surface that survives the **late-summer ablation minimum across
multiple years**: classify each clear look as snow / not-snow by NDSI over a late-summer window,
then require persistence across years. A single-date snow map cannot separate perennial snow from a
late seasonal snowpack; only persistence can.

**Independently designed, then confirmed against the literature.** This persistence approach was
built from first principles before any literature review. A subsequent check for credible precedent
found the USGS persistent-snow method (Selkowitz & Forster 2016) to be strikingly similar — the same
late-summer window, per-scene NDSI classification, and multi-year persistence requirement, validated
there against the Randolph Glacier Inventory. The convergence cross-validates the design; it was not
its source.

---

## 2. Reflectance screens before NDSI (MODIS/VIIRS ATBD)

Every look is screened before the NDSI is computed: reflectance clipped to `[0, 1]`, both bands
required `> 0`, and a low-VIS screen (`green > 0.10`).

**Why:** naive NDSI thresholding produced values far outside the mathematical `[-1, 1]` range
(observed −6446 to +2552). Two causes:
- unmasked nodata (Sentinel DN 0 → −0.1 reflectance after the offset) fed the ratio;
- Sen2Cor returns over-unity reflectance on bright snow.
Both detonate `(g-w)/(g+w)`. The screens are the operational MODIS/VIIRS snow-ATBD conditions,
so the fix is also a citable alignment, not an ad-hoc clamp. The low-VIS screen additionally
rejects dark-surface false snow (very low reflectance can give NDSI > 0 on non-snow).

---

## 3. Multi-sensor fusion (Sentinel-2 + Landsat 8/9)

All three sensors feed a shared 30 m grid as independent per-(tile, date, sensor) acquisitions.

**Why:** a single sensor (Sentinel-2 alone) leaves the ablation-minimum window cloud-starved in
some years — coverage was especially thin in 2022 and 2024. Landsat's orbit gives clear-sky days
*independent* of Sentinel's, filling those gaps. The grid is 30 m (Landsat-native, DEM-matched);
Sentinel is **downsampled** by area-averaging rather than upsampling Landsat, so no detail is invented.

**Front-end differs per sensor, logic converges after:** Sentinel uses SCL integer classes,
Landsat uses bit-packed QA_PIXEL; each has its own reflectance scaling. After producing true
reflectance and a boolean cloud/water mask, NDSI and the snow/not-snow/masked/water logic are identical.

---

## 4. NDSI overrides the cloud mask on snow pixels

A pixel flagged cloud is kept if it reads NDSI-snow.

**Why:** both Sentinel SCL and Landsat CFMask systematically over-flag bright snow as cloud
(Landsat's docs state this explicitly — residual ice on warm ground gets called cloud). Without
the override, the cloud gate rejects exactly the snow the pipeline exists to find. NDSI checks the
one band that actually separates snow from cloud (SWIR: dark for snow, bright for cloud), so it is
the correct arbiter when the two disagree.

---

## 5. Bare is a **yearly aggregate**, never a per-epoch veto

A pixel's per-year verdict comes from `snow_frac` = the fraction of its clear *land* looks that were
snow; a single not-snow look does not disqualify it. **snow-that-year** if `snow_frac ≥ 0.80`;
otherwise **bare-that-year**; a year abstains entirely if it has too few looks to trust the verdict
(see #5b).

**Why — and what was tried first (this was the single most important correction):**
- **Tried:** per-epoch veto — mark the year bare if *any* clear look is not-snow, and require snow
  in *every* clear look to call it perennial. **Result:** the map emptied out. Over 15–40 looks a
  year, a genuinely perennial pixel *always* catches a stray not-snow look — thin cloud dropping
  NDSI below threshold, a look taken far from the ablation minimum, a mixed edge pixel. Demanding perfection across all
  looks disqualifies essentially every real pixel.
- **Fixed:** decide the year on the *fraction* of clear looks that are snow, tolerating a noise
  minority. Bare becomes a property of the year, not of any single observation.

---

## 5b. No snow-fraction "dead zone"; asymmetric look-count floors

The per-year verdict has **one** boundary (the 0.80 snow bar), not two. Below the bar a pixel is bare
(given enough looks) or abstains (too thin) — there is no neutral middle band.

**Why — and what was tried first:**
- **Tried:** a symmetric pair of thresholds — snow if `snow_frac ≥ 0.80`, bare if `snow_frac ≤ 0.20`,
  leaving a 0.20–0.80 "dead zone" as neither. **Result (a real bug):** dead-zone years still counted as
  *decided* (raising the decided-year count) while contributing **zero** to confidence, so they silently
  dragged a pixel's mean confidence down and biased it toward not-perennial. The dead zone was never
  neutral; it was a hidden anti-perennial pull. It was legacy code from when snow and bare were treated
  symmetrically (before #6's asymmetry).
- **Fixed:** a year is *decided* only if it reaches a verdict (snow or bare). Below 0.80 is bare; a
  mid-fraction year that is too thin to trust simply abstains and is excluded from both the decided-year
  count and the confidence mean. Genuinely neutral.

**Asymmetric look-count floors** decide when a fraction is trustworthy:
- **Snow** needs `≥ MIN_SNOW_LOOKS` (2) *actual snow looks* — gating on snow-evidence count, not total
  looks, so a thin-but-unanimous pixel (e.g. 2/2 snow) is not punished for sparse coverage while a
  single-look-snow pixel (possible noise) is held back.
- **Bare** needs only `≥ BARE_MIN_LOOKS` (2) total looks — snow's *absence* is unambiguous, so it needs
  no noise-tolerance headroom.

An earlier attempt gated snow on *total* looks (`nvalid ≥ 5`); this over-penalised data-scarce years and
collapsed PERENNIAL (a monsoon-clouded pixel can be genuinely perennial without ever getting 5 clear
looks in a year). Gating on snow-look count fixed it.

> These look-floors are **pragmatic, uncalibrated** heuristics. No per-look snow-miss error rate exists
> in the literature to calibrate them against (published work reports scene-level accuracy, not
> per-observation miss probability), and there is no in-situ Beas Kund data. They are engineering
> choices, stated as such.

---

## 6. Asymmetric evidence: not-perennial is easier to prove than perennial

- **PERENNIAL** needs the full bar: `≥ MIN_YEARS` decided years, high ablation-minimum-weighted confidence,
  **and** bare in `≤ MAX_BARE_YEARS` years.
- **NOT-PERENNIAL** fires on bare in `> MAX_BARE_YEARS` years **regardless of total decided years**
  (no MIN_YEARS requirement).

**Why — and what was tried first:**
- **Tried:** symmetric rule — require `≥ MIN_YEARS` decided years to call a pixel *anything*.
  **Result:** pixels clearly bare in 3 years but with only 3 decided years were dumped into
  UNDECIDED, even though "bare 3 years running" already rules out perennial.
- **Fixed:** perennial is a persistence claim and needs multi-year evidence; ruling perennial *out*
  needs only positive bare evidence. So not-perennial can be decided on fewer years. This is the
  same asymmetry applied at the year level that #5 applies at the epoch level.

Also rejected earlier: a **record-wide "ever bare" veto** (disqualify if bare in any single decided
year) — same failure mode as #5 one level up, one anomalous or thin year killed real perennial
pixels. Replaced by the `≤ MAX_BARE_YEARS` tolerance.

---

## 7. Ablation-minimum timing weighting

Each snow look contributes a weight that peaks on the ablation-minimum plateau and decays for
too-early (incomplete melt) or too-late (fresh-snow risk) last-looks. Per-pixel confidence is the
best (highest-weighted) snow look within a year; the multi-year confidence is the mean across
decided years.

**Why:** recency matters *within* a year — a snow look at the seasonal minimum is stronger evidence
of perennial snow than one in early August. This is an extension of the Selkowitz-style persistence
(which weights all in-window looks equally); it is introduced because this AOI's ablation window is
cloud-scarce, so off-peak looks (far from the ablation minimum) must be admitted but down-weighted rather than trusted equally.

linear decrease outside the ablation-min window, floored at 0.20.

---

## 8. Ablation-minimum window is grounded per-AOI (not a fixed date)

**Beas Kund (monsoon Himalaya):** late Aug – mid Sept. Grounded in (a) this project's own pooled
snow-fraction-vs-day-of-year curve (bottoms ~30 Aug – 1 Sept) and (b) the same-basin Beas River
Basin MODIS/ERA5 study (August SCA minimum).

**Why not a generic "western Himalaya" citation:** the ablation minimum is climate-regime specific.
An arid trans-Himalayan valley (e.g. Zanskar) has a different phenology than a monsoon-facing basin
and was **not** used, despite being nearby — the monsoon drives Beas Kund's August minimum.

**Mont Blanc (validation AOI):** mid-to-late September, cited from Alpine glaciology literature
(the Alpine minimum is *later* than the Himalayan one). Reusing the Himalayan window in the Alps
would test snow presence too early — while seasonal snow is still melting — and **over-claim**
perennial. The window must sit at each AOI's true minimum for "snow present in the window" to mean
"snow survived the annual minimum."

---

## 9. Every non-verdict pixel gets a **reason**, not a shrug

Five outcomes instead of a single "undecided":
- **NO_DATA** — the trusted masks (cloud / shadow / off-tile) rejected *every* look (`ndec == 0`).
  This is a data-coverage statement, not indecision.
- **WATER** — a known water surface (SCL/QA water), excluded from the snow question entirely.
- **UNDECIDED** — genuinely observed but too thin (`0 < ndec < MIN_YEARS`) and not bare enough to
  rule perennial out.
- **NOT_PERENNIAL / PERENNIAL** — real verdicts.

**Why:** collapsing "the masks couldn't see it" and "we saw it but it's ambiguous" into one label
hides the difference between a coverage limit and a genuine uncertainty. Water especially is a
*known* surface — calling a tarn "no-data" throws away a positive identification. Separating them
makes the mask an honest statement of exactly what is known, and why, at every pixel.

---

## 10. Mask uses **all 8 years**; the change study uses **4**

These are deliberately different year sets because they answer different questions.

**Mask (2017–2024, all 8 years).** Thin early years (2017–18) are *load-bearing*: they push ~9.7k
pixels over the `MIN_YEARS` bar. Dropping them **loses** decided pixels (dropping years can only
lower a pixel's decided-year count). More years = more evidence, so keep them all.

**Change study (2019, 2022, 2023, 2024).** See #11.

---

## 11. Interannual change measured on a **fixed observable intersection**

The change study counts perennial pixels **only within a fixed set of pixels that have a real binary
verdict in every study year.**

**Why the intersection is essential — the core of the change design:**
change can only be trusted if the *same ground* is measured every year. If instead each year's count
came from whatever pixels happened to be cloud-free that year, then year-to-year differences would
reflect *what we could see* (sampling drift) rather than *what changed on the ground*. Holding the
pixel set fixed removes that confound by construction: any rise or fall in the count is real change
on that fixed ground.

**Why the study years differ from the mask years (and why we dropped some):**
- **Tried:** the full 8-year intersection. **Result:** ~2.2k pixels — the thin years (2017/18/20/21)
  each require a verdict everywhere, and almost no pixel is decided in *all* eight. Statistically useless.
- **Fixed:** drop the thin years. Dropping 2017/18/20/21 *grows* the intersection to ~33k pixels
  (2019, 2022, 2023, 2024), because fewer required years means more pixels clear the all-years bar.
  So for the mask, dropping years loses pixels (#10); for the change intersection, dropping the thin
  years *gains* pixels. Opposite directions, different questions.

**Also rejected for the change study:**
- **Confidence deltas (decimal change).** Differencing ablation-minimum-weighted confidence values across years
  conflates real change with measurement variance — different sensors and acquisition timing give
  different confidence *values* for the *same* physical state. The verdict unit is **binary**
  (perennial-that-year yes/no) instead.
- **Two-period mean-of-means** (early-half vs late-half average). Blends good and bad years into a
  number that doesn't cleanly mean anything; not a real trajectory.
- **Self-normalized fraction** (perennial / observable per year). The *composition* of the observable
  set changes year to year in a terrain-correlated way, so the fraction is a biased sample even when
  self-normalized. The fixed intersection controls this; the fraction does not.

**Scope:** the fixed set is ~25% of the AOI. Observation is uneven (persistent
cloud, terrain shadow, sensor gaps), so this is a trend on the reliably-observed subset, **not**
basin-wide, and no aspect-differentiated (north- vs south-slope) melt claim is made.

Since a fixed set of pixels is used for comparison that have definitive stance of bare or perennial snow 
throughout years, it can result in spatial bias, for example north-facing or deeply-shadowed slopes are 
observed less because of masking, and different aspects melt at different rates (sun-exposed slopes melt 
faster than north-facing ones in the northern hemisphere)

---

## 12. Validation against an operational product (Theia LIS SCD)

The method is cross-validated against the **Theia Sentinel-2 Snow Cover Yearly Synthesis (LIS L3B)**
snow-cover-duration (SCD) product over the Mont Blanc massif, 2019–2024.

**Why Mont Blanc, not Beas Kund:** no operational snow product covers the Himalayan AOI, so the
*method* is validated where a reference exists (the pipeline is AOI-agnostic — swap bbox + UTM zone),
then argued to transfer.

Perennial is defined as **SCD ≥ 360 days** (snow essentially the entire hydrological year) — a
*definitional* threshold, not one chosen to maximise agreement (choosing by best fit would be
circular: letting the reference's agreement define the reference). Agreement is checked across the
whole SCD range and is high throughout, so the choice isn't fit-driven.

**Result (at SCD ≥ 360):** Cohen's kappa 0.86, precision 0.94; high and stable (0.83–0.87) across
SCD 330–360. The method does not over-detect relative to the reference.

**Every disagreement pixel is explained — none is classification error** (the comparison is over
decided pixels only):
- **Ours-only** (~11,100 px, low elevation ~2700 m): **glacier tongues** (Mer de Glace, Argentière),
  visible as coherent downslope features. Both products detect snow/ice by NDSI, so this is **not** a
  detection difference — it is the one documented algorithmic difference: Theia's LIS excludes snow
  below a per-scene **DEM snowline elevation** (Gascoin et al. 2019); glacier tongues descend below it.
  The method has no snowline gate, so it maps the full snow-and-ice cryosphere extent.
- **Theia-only** (~4,600 px, high elevation ~3220 m): a limitation on the *method's* side, not evidence
  Theia over-includes. ~88% are low-confidence (snow seen across enough years but below the confidence
  bar — conservatism); ~12% correlate strongly with north-facing, steep, shadowed terrain (northness
  +0.62 vs +0.25, 33° vs 25°) where optical NDSI snow mapping is hardest. We report the aspect
  *correlation*; we do not claim a specific radiometric mechanism.

---

## Localisation notes (revisit for wide-area / operational use)

- `MAX_BARE_YEARS = 2` is an absolute count, defensible for this 8-year study (2/8 = 25%). For runs
  with different year counts it should become a **fraction** of decided years for scale-invariance.
- Thresholds (`NDSI_SNOW`, `YEAR_SNOW_FRAC`, `PERENNIAL_W`, the ablation-minimum window, the look-floors) are
  tuned for these AOIs and would need re-grounding for a different climate regime.
- **Known limitation (surfaced by validation):** the method slightly under-detects perennial snow on
  north-facing, steep, shadowed slopes, where the disagreements with Theia concentrate. This is the
  same aspect-linked observational limit noted for the change study; correcting it (illumination
  normalisation, shadow-aware detection) is future work.
- **Operational hardening deferred:** same-date multi-product merging and cross-platform temporal
  fusion (as in the LIS `snow_annual_map` pipeline) are not implemented; they matter for wide-area,
  many-tile runs, not for this single-AOI study.

---

## Confidence gate: a data-quality filter, not an arbitrary threshold

The multi-year confidence (mean ablation-minimum-weighted per-year confidence) is not just an output layer — it
gates the final PERENNIAL verdict (conf ≥ `PERENNIAL_W`, 0.75). At Beas Kund it is the dominant
filter: of ~25.8k pixels that pass the persistence structure (enough decided years, snow-fraction, bare
tolerance), the confidence gate rejects 74%, leaving ~6.7k PERENNIAL.

Why this is correct behaviour, not over-rejection. Confidence is driven by observation timing —
a pixel scores high only if its snow looks land near the ablation minimum. Under sparse, cloudy coverage
(Beas Kund, monsoon-influenced), many genuinely-persistent pixels are only ever seen snow off-peak (far from the ablation minimum in time),
so they score low and are correctly held back rather than over-claimed. The gate is the mechanism by
which the mask declines to assert perennial status on poorly-timed evidence.

Demonstrated across two AOIs. If the aggressiveness were a scarcity signature, a data-rich AOI
should be far less threshold-sensitive. It is:

`PERENNIAL_W` 0.5 → 0.8	perennial-count change
Beas Kund (sparse, monsoon)	4.6× (21,209 → 4,623)
Mont Blanc (dense, Alpine)	1.3× (96,953 → 76,036)

Where coverage is dense, confidence saturates and the threshold is near-inert; where sparse, it
discriminates. So the threshold's influence is itself a diagnostic of data quality. A consequence:
the Mont Blanc validation (kappa 0.87, run at 0.75) is robust to the threshold — 0.70/0.75/0.80
give 82.6k/81.5k/76.0k pixels — so the headline agreement is not an artifact of the chosen cut.

Rejected: calibrating `PERENNIAL_W` to maximise agreement with Theia. Theia could structurally omits glacier-tongue ice that this method captures because glaciers can travel further below into valleys than perennial snow and Theia's algorithm explicitly thresholds snowline by elevation in the first pass, so calibrating to it
would tune the threshold to reproduce the reference's blind spot. 