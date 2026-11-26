# Middletown Lead Blast — project brief

Personal project. A public, plain-language web app about the Nov. 25, 2026 lead azide detonation at the Iowa Army Ammunition Plant (IAAAP), backed by my own dispersion analysis. Goals: (1) explain the event to Iowa residents who are not scientists, (2) build skills in dispersion modeling, cloud-native data, and browser map rendering.

Design prototype (Claude Design canvas): https://claude.ai/artifact/G1r8Uw8JWGVsKty4yoWvAq

## Working conventions

- Python unless stated otherwise. Docstrings: numpy style, short.
- Keep answers and comments direct and factual.
- Every number shown in the app must trace to a source or to a documented calculation.
- No code or data from this project implies official guidance. Keep the disclaimer.

## Event facts (verify before publishing)

| Item | Value | Source |
|---|---|---|
| Detonation time | 2026-11-25 15:00 UTC (09:00 CST), two detonations in quick succession | Wikipedia, NYT |
| Material | ~4,300 kg (~9,500 lb) lead azide, Pb(N3)2 | Wikipedia |
| Lead mass | ~3,060 kg (~6,750 lb); Pb is 71.1% of lead azide by mass | Calculated |
| Donor charge | ~3,000 lb C-4 | Wikipedia |
| Visible cloud | Several hundred feet; gone within ~6 min | Wikipedia, NYT |
| Wind | From NE toward SW at release | Wikipedia |
| Reported drift | South (press accounts) — differs from the SW forecast axis | Wikipedia |
| Mitigation | Sand barriers, lime fill, saturated soil, misters | Wikipedia |
| Plant centroid | ~40.79 N, -91.245 W (approximate; actual blast point within the ~19,000-acre site unknown) | Mapped |
| Official advice | Iowa HHS: stay indoors, windows closed 24–48 h; wet-clean outdoor surfaces | Wikipedia |
| EPA public line | 1-833-649-9624 | EPA release |

Airborne-fraction cases in circulation: 7% (low), 26% (U.S. Army "mitigated"), 55% (high), 100% (EPA permit). The Army's deposition maps use 0.26.

## Data sources

### EPA real-time lead monitors (primary observational dataset)

- Dashboard: https://experience.arcgis.com/experience/1ddcf8b9d8314971b00dac8893ae9cd6
- Instruments: Xact (continuous XRF metals monitors). Reporting interval: 5 min.
- Feature services (public ArcGIS REST; page with `resultOffset`, `f=geojson`):
  - Xact results: `https://services.arcgis.com/cJ9YHowT8TU7DUyn/arcgis/rest/services/IAAAP_Xact_Data_Public_Viewer/FeatureServer/0`
  - Sampling locations (likely lab-sample sites): `https://services.arcgis.com/cJ9YHowT8TU7DUyn/arcgis/rest/services/Air_Sampling_and_Monitoring_Location/FeatureServer/0`
- Fields: `objectid, globalid, Mon_Date_Time (epoch ms, UTC), Latitude, Longitude, Mon_Results, Mon_Units (µg/m3), Air_Station_ID`
- Stations:

| ID | Lat | Lon | Approx. distance / bearing from centroid |
|---|---|---|---|
| AS01 | 40.77694 | -91.32317 | ~7 km (4 mi) WSW |
| AS02 | 40.708378 | -91.449928 | ~20 km (12 mi) SW |
| AS03 | 40.646668 | -91.563951 | ~31 km (19 mi) SW |
| AS04 | 40.583555 | -91.697377 | ~45 km (28 mi) SW |

- Caveats: all four stations lie on one line along a ~240° bearing (the forecast axis). Values sampled so far are exact `0` with no non-detect flag, detection limit, or pollutant field. Detection limit at the 5-min interval is unknown; request it from EPA. Sampled records so far covered 13:00–13:40 UTC only; pull the full series.
- **Archive both layers soon.** Response dashboards change or disappear.

### Weather

- HRRR (3 km, hourly analyses, WRF-ARW based): `s3://noaa-hrrr-bdp-pds` (no account needed).
- HRRR, RAP, NAM 12 km in HYSPLIT (ARL) format: NOAA ARL archives (READY site).
- Validation obs: DVN (Davenport) 12 UTC sounding, KBRL (Burlington) ASOS.
- Satellite: GOES-19 ABI, `s3://noaa-goes19`, 1-min mesoscale / 5-min CONUS, frames 15:00–15:10 UTC. Python: `goes2go`. Can show plume geometry if skies were clear; cannot detect lead.

### Other

- Army deposition maps and updates: Illinois EPA page https://epa.illinois.gov/iowa-plant-updates.html
- Iowa Lead Tracker (independent, Gaussian puff, forecast-driven, MIT code / CC BY 4.0 content): https://github.com/iowaleadtracker/iowaleadtracker — see `MODEL.md` and `scenarios/iowa-aap-lead-azide/` for source-term assumptions to reuse.
- PurpleAir: API is points-based; new users get 1M points (enough here). PM2.5 useful for plume timing and as an upper bound on total fine PM (Pb ≤ PM). Apply EPA (Barkjohn) humidity correction. Check sensor coverage S/SW of the plant first.
- EPA lab results (air, soil, water): pending; watch EPA Region 7 pages.

## Science approach (decided)

**Model: HYSPLIT on HRRR, not WRF.** Source-term uncertainty (1–2 orders of magnitude) dominates; terrain is flat; HRRR analyses are observation-constrained. WRF + FLEXPART-WRF only if HRRR fails validation, or later as a separate skill-building track.

**Core question:** did the plume pass over the four EPA monitors? If yes, their non-detects plus the detection limit give an upper bound on the airborne lead fraction, testable against the Army's 26%. If no, "zero detection" carries little information.

**Unit-source design:**
- One HYSPLIT run per size bin × release height, unit mass (1 kg) each.
- Size bins (µm): 0.3, 1, 3, 10, 30. Particle density ~9.5 g/cm³ (PbO).
- Release: volume source from ground to cloud top; tops ~150, 300, 500 m; duration ~2 min starting 15:00 UTC.
- Airborne fraction and size distribution (lognormal, MMD/GSD) become weights applied in post-processing and in the browser.
- Weather ensemble: repeat on HRRR, RAP, NAM.
- Outputs: concentration and dry/wet deposition, 0–36 h.

**Settling velocities (PbO, Stokes + Cunningham):**

| Diameter | v_s | Time to fall 300 m |
|---|---|---|
| 30 µm | ~26 cm/s | ~20 min |
| 10 µm | ~3 cm/s | ~3 h |
| 3 µm | ~0.3 cm/s | ~30 h |
| ≤1 µm | ~0.03 cm/s | stays aloft |

Coarse fraction = near-field problem (first hour, high-res wind matters). Fine fraction = far-field problem (boundary-layer growth and mixing matter). Release was into a likely stable morning boundary layer.

## Plan

1. **Archive EPA data.** Pull full Xact series and sampling-location layer to GeoJSON/Parquet with timestamps.
2. **Quick look.** READY/HYSPLIT single run on HRRR; compare direction to GOES frames and the reported southward drift.
3. **Validate met.** HRRR vs DVN sounding and KBRL winds around 12–18 UTC.
4. **Unit-source runs.** Local HYSPLIT, 15 runs × 3 met datasets. Convert output to Zarr (xarray).
5. **Monitor analysis.** Modeled concentration time series at AS01–AS04 per scenario; plume-path overlap; bound on airborne fraction once the detection limit is known.
6. **App v1.** Story, timeline, monitors, conditions, GOES frames (no model needed).
7. **App v2.** Model explorer with sliders (client-side weighting of unit fields).
8. **App v3.** EPA lab results vs model; PurpleAir overlay; source inversion if data allow.

## App architecture

Fully static. No backend.

| Job | Tool |
|---|---|
| Framework | React + Vite (or Next.js static export) |
| Base map | MapLibre GL JS |
| Basemap tiles | Protomaps PMTiles on R2 |
| Layers | deck.gl |
| Zarr in browser | zarrita.js |
| Charts | Observable Plot |
| Scroll narrative | Scrollama |
| Data prep | xarray, zarr, rioxarray (COGs), ndpyramid only if needed |

- Hosting: frontend on GitHub Pages or Cloudflare Pages; data on Cloudflare R2 (10 GB free, no egress fees; set a CORS rule for browser Zarr reads). Avoid AWS (old free tier expired; egress charges).
- Snapshot EPA data to R2 rather than querying their service live.
- Model explorer v1: weighted sum in JS typed arrays → BitmapLayer. v2 (skill-building): weighted sum in a custom deck.gl shader, unit fields as textures, slider weights as uniforms.
- Estimated data size: well under 1 GB.

## Design spec

Audience: Iowa residents, non-scientists. Plain language, US units (lb, miles, feet), Central time.

**Page order:**
1. Dark header band: site name, nav, title "Where did the lead go?", one-paragraph intro. Keep compact.
2. Disclaimer banner: unofficial personal project; can't say whether a home or yard is safe; EPA public line.
3. Four quick-answer cards: What happened · What the monitors found · What we don't know yet · Official advice.
4. Map + "what if" sliders: airborne fraction (Army 26% marked), dust fineness (smoke-fine → gritty), cloud height (ft); time slider in CST; result card with lb of lead airborne and "Would EPA's monitors have caught it?"
5. Air monitors: four stations by distance in miles, "No lead detected" badges, plain explanation of what "zero" means and the south-vs-southwest gap.
6. Why dust size matters: three cards (coarse / fine / smoke-fine) with hair-width comparisons.
7. Timeline: 2004 → March 2026 → May 2026 → Nov 2026 lawsuit → Nov 25 blast → EPA statement → lab results pending.
8. Footer: about, links (methods for scientists, Iowa HHS, EPA data), "Last updated" date (auto-fill from build/data refresh).

**Palette (from nicolekeeney.com):**

| Role | Hex |
|---|---|
| Dark band / dark cards | #0D1116 |
| Page background | #FFFFFF |
| Body text | #272822 |
| Secondary text | #33414E, #5A6672 |
| Dust estimate (fills) | #E36588 |
| Dust accent text, slider values, dates | #9A275A |
| Monitors (dots, badges) | #1D5A8F |
| Links | #2E7DB8 (hover #1D5A8F) |
| Disclaimer banner | bg #FCEEF2, border #F2C1CF, text #7A1F47 |
| Borders | #E3E6EA, #EEF0F3 |
| Map land / river | #EEF2F6 / #A9C4DB |

Colorblind rule: dust (pink/magenta) vs monitors (dark blue) must differ in lightness, not hue alone. No red/green pairs.

**Fonts:** Courier Prime (headings, site name), Lato (body).

**Accessibility:** real `<button>`, `<label>`, `<input>`; text contrast ≥4.5:1; touch targets ≥44 px; works at phone width.

## Open questions

- Xact detection limit at the 5-min reporting interval (ask EPA).
- Exact blast coordinates within the plant.
- Whether precipitation fell in the 48 h after release (wet deposition).
- Whether EPA will publish lab results with locations, times, values, and detection limits (needed for inversion).
- PurpleAir sensor coverage south and southwest of the plant.
- Whether resuspended legacy contamination (Superfund site, depleted uranium) is in scope.
