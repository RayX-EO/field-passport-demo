# Field Passport API v2

Per-field agronomic report for UK winter wheat from the 2018–2025
satellite yield product, plus the live in-season stream for the current
season. Send a field boundary (or a farm's worth), receive: achievable
yield, the gap and its causes, within-field structure, weather risk,
national ranking, and this season's classification + yield forecast.

```
POST https://c8e5gkjyia.execute-api.eu-west-2.amazonaws.com/api/v2/passport
```

Latency: ~0.4–0.55 s warm, ~1 s cold. Batch: a 15-field farm returns in
~2 s — all fields' data reads are merged into one deduplicated fetch, so
a batch costs barely more than a single field.

> `/api/passport` (v1) is deprecated — sunset 2026-09-30, then `410`.
> Integrate against v2 only.

## Authentication

Shared secret in a header on every request:

    X-Api-Token: <token>

Missing/wrong token → `401 {"error": "unauthorized"}`. Treat the token as
a credential: server-side only, or supplied by the user at runtime — never
embedded in a public page.

## Request

`Content-Type: application/json`. Body = GeoJSON: a `Polygon` or
`MultiPolygon`, bare geometry or wrapped in a `Feature` /
`FeatureCollection`.

- **CRS**: EPSG:4326 (lon/lat) or EPSG:27700 (British National Grid),
  auto-detected — values that look like lon/lat are treated as lon/lat.
- **One management unit per polygon**, ≤ 1000 ha (else `400`). A boundary
  straddling several real fields degrades the within-field statistics, and
  the ≥ 80 % cover gate will exclude seasons.
- **Coverage**: Great Britain winter-wheat area, harvests 2018–2025 (+ the
  current season via `in_season`). A polygon outside wheat country returns
  a valid, gated report — see `products.status`.

```bash
curl -X POST "$BASE/api/v2/passport" \
  -H "Content-Type: application/json" -H "X-Api-Token: $TOKEN" \
  --data @field.geojson
```

### Batch — one farm, one request

A `FeatureCollection` with N features returns N passports in one response:

```json
{"n_fields": 15, "fields": [ { ...passport, "properties": {...} }, ... ]}
```

- Limits: **≤ 40 features and ≤ 2000 ha total** per request (else `400`).
- Feature `properties` are echoed back verbatim onto each report — put
  YOUR field key there (see `field_id` below for why).
- A single feature (or bare geometry) returns the passport object alone,
  unwrapped.

### Query parameter: `?weekly=`

Controls the in-season weekly stream (applies to every field in a batch):

| value | behaviour |
|---|---|
| `headline` (default) | last ~3 weeks per product fetched — verdict, forecast and forecast grid are identical to `full`; `trajectory` holds just those weeks and `trajectory_note` says so |
| `full` | the whole season's weekly trajectory (~0.1–0.4 s extra) |
| `off` | skip the stream entirely; `detail.in_season` is `null` |

Anything else → `400 {"error": "weekly must be off|headline|full"}`.

## Response (200)

| key | meaning |
|---|---|
| `field_id` | **a display label, NOT a key**: 6 hex chars (~24 bits; sha256 of the 1 mm-quantised BNG rings), so collisions become likely around a few thousand fields. Cache and join on YOUR OWN geometry version — that's what the echoed batch `properties` are for |
| `physical_identity` | `area_ha`, `centroid_bng [E,N]`, boundary provenance note |
| `products` | the four headline products + ranking (below) — build UIs from this |
| `detail` | every underlying number (below) |
| `engine` | `"rust"` |

### `products.status` — evidence is a gate, not a badge

`"ok"` or `"no_wheat_history"`. When the polygon has **zero full wheat
seasons** (e.g. permanent pasture), measured products are WITHHELD, not
synthesised: `gap` and `ranking` come back `null` with a `withheld`
reason, and `potential` / `climate_risk` carry
`is_location_attribute: true` — they describe the LOCATION (what nearby
wheat achieves, the local weather climate), not this field. Render them
separately and labelled as such; never headline a field that has never
grown wheat. Season-level means are also support-gated:
`season_detail.<yr>.mean_yield_t_ha` is `null` below 30 valid pixels
(cover and pixel counts stay visible for transparency).

### `products` — the categorical, user-facing layer

Class words are positions in national distributions over GB wheat fields
(terciles unless stated), never fixed judgements.

| block | fields |
|---|---|
| `potential` | `attainable_yield_t_ha` (local 5.85 km p95 frontier, best season), `class` low/moderate/high, `percentile_gb`, `national_range_t_ha` [p10,p90] — NB the national spread is only ~0.6 t/ha: read the gap, not this class |
| `gap` | `total_gap_t_ha`, `class` small/moderate/large, `percentile_gb`, `dominant_cause`, `causes_t_ha` {weather, soil, rotation, terrain, field_edges, other_management}, `overlap_scaled` (true when marginal sub-causes were scaled to fit G_M) |
| `within_field` | `class` uniform/moderate/variable, `spread_t_ha`, `percentile_gb`, `addressable_spread_t_ha` + `addressable_share` (upper bound management could level), `areas_ha` {high_capacity_stable, persistently_constrained, season_sensitive, mixed} |
| `climate_risk` | `class` low/moderate/high (realised 2018-25 weather gap vs GB), `outlook` {chance_of_costly_season, typical_annual_loss_t_ha, bad_season_loss_t_ha, bad_season_example, n_seasons}, `dominant_hazard` dry/wet/hot (+ its GB percentile), `exposure_percentile_gb`, `response_to_dominant_hazard` |
| `ranking` | `vs_neighbours_20km`, `vs_similar_fields_gb` (k=256 analogue fields by terrain/soil/climate), `class` bottom/middle/top third. Percentile = share of comparators with a LARGER persistent gap — **higher is better** |
| `confidence` | high (≥ 4 full seasons) / moderate (≥ 2) / low |
| `evidence` | human-readable season count |

### `detail` — the full numbers

`season_detail` (per year: `mean_yield_t_ha`, `cover_of_polygon`,
`n_valid_px`, `eo_wheat_season`, `full_season`) · `gap_by_season` (G_M,
G_Σ, G_W, total per full season) · `yield_gap` (telescope terms +
rotation split) · `productive_memory` (expected weather-normalised yield,
best season, ceiling, percentiles, stability) · `within_field_variation`
(ha classes + `class_grid`) · `within_field_split` (observed vs
soil-expected spread) · `field_edges` (outer 20 m vs interior beyond
40 m) · `terrain` · `rotation` (position + delta per season) ·
`climate_behaviour` (typed dry/wet/hot responses, risk block) ·
`crop_history` (2016–2025, LCCM majority class) · `gap_map` (below) ·
`in_season` (below) · `evidence`.

### `detail.in_season` — this season's weekly stream

Live weekly products for the CURRENT season, independent of the 2018–25
archive: a 5 m winter-wheat classification of the published field map
and, from July, a 10 m in-season yield forecast.

```json
"in_season": {
  "season": 2026, "as_of": "2026-08-17",
  "classification": {"resolution_m": 5, "source_week": "2026-08-17",
    "wheat_share_of_polygon": 0.94, "p_wheat": 0.91,
    "verdict": "winter wheat"},
  "forecast": {"resolution_m": 10, "as_of": "2026-08-17",
    "yield_mean_t_ha": 9.1, "yield_p05_t_ha": 7.4, "yield_p95_t_ha": 10.6,
    "n_px": 812,
    "grid": {"resolution_m": 10, "scale": 0.01, "nodata": 0,
             "w": 42, "h": 38, "transform": [10,0,577930.0,0,-10,299710.0],
             "data": [0, 912, 887, "..."]}},
  "trajectory": [ {"monday": "2026-03-30", "p_wheat": 0.88}, "..." ],
  "trajectory_note": "latest weeks only — request ?weekly=full ...",
  "caveats": ["..."]
}
```

- `verdict`: `winter wheat` (share ≥ 0.6), `not winter wheat` (≤ 0.2),
  else `mixed`.
- `forecast.grid` is the latest week's **per-pixel** forecast raster:
  row-major ints, `value × scale` = t/ha (912 → 9.12), `0` = nodata
  (outside the polygon, or not classified winter wheat this season).
  `forecast` is `null` (never guessed) when the field isn't wheat this
  season or the forecast hasn't come online yet.
- `trajectory` carries every fetched week with any data, so a UI can show
  the season converging. Weeks appear ~every Monday; a week's numbers can
  be revised if it is re-published.
- The whole block is `null` when the weekly stream has no data for the
  polygon, or with `?weekly=off`.

### Map-layer grids

All spatial layers ship as banded or valued **grids** for client-side
rendering:

```json
"gap_map": {
  "labels": ["< 0.5", "0.5–1.0", "1.0–1.5", "1.5–2.0", "≥ 2.0"],
  "grid": {"data": [0,1,3, "..."], "w": 128, "h": 96,
           "transform": [2.5, 0, 577800.0, 0, -2.5, 299850.0]},
  "mean_gm_t_ha": 1.7, "share_over_1_t_ha": 0.77,
  "performance": { "same shape — share-of-achievable bands": "..." }
}
```

- `data` is row-major; `0` = no data. Banded grids (`gap_map`,
  `performance`): `1..5` indexes into `labels`.
  `within_field_variation.class_grid` (10 m): `1` = persistently
  constrained, `2` = high-capacity stable, `3` = season-sensitive.
  `in_season.forecast.grid`: scaled values (see above).
- `transform` is the affine `(a,b,c,d,e,f)` in **EPSG:27700**, north-up
  in the British National Grid.
- **Rendering warning — BNG grid north is not map north.** The meridian
  convergence rotates BNG grids ~1–2° from true north over England (worse
  eastward): pasting the raster into an axis-aligned lat/lng image
  overlay visibly shears it against the field boundary (~8 m at the
  corners of a mid-size field). Either reproject per pixel, or fit the
  BNG→WGS84 affine on three projected grid corners and inverse-sample per
  destination pixel — that affine is exact to well under a pixel at field
  scale. Our reference implementation is `gridLayer()` in the demo pages.

## Errors

| code | body | cause |
|---|---|---|
| 400 | `{"error": "bad geojson: ..."}` | unparseable body / geometry |
| 400 | `{"error": "polygon larger than 1000 ha — ..."}` | per-polygon cap |
| 400 | `{"error": "... batch limit ..."}` | > 40 features or > 2000 ha total |
| 400 | `{"error": "weekly must be off|headline|full"}` | bad query param |
| 401 | `{"error": "unauthorized"}` | token missing/wrong |
| 500 | `{"error": ...}` | server fault — report it, never retry-loop |

## Interpretation caveats (contractual, not decorative)

1. Yields are satellite retrievals: ~±1 t/ha on any single field-season;
   patterns across seasons/space are more reliable than any single number.
2. `other_management` / G_M is an **upper bound** on management — it also
   contains variety, drainage, sub-map-resolution soil and retrieval
   error.
3. Weather odds rest on **8 observed seasons** — rough odds, not
   forecasts, and not a climate-change projection.
4. Class words are national-distribution positions; `potential.class` in
   particular carries little signal (see its row above).
5. In-season products stabilise through the season: classification firms
   up through spring, the forecast narrows toward harvest. They are
   EO-derived, not declared cropping.
6. Crop history is EO-classified (UKCEH LCCM), not declared cropping, and
   is licence-restricted — do not redistribute it raw.

## Versioning

Breaking response changes ship as a new path (`/api/v3/...`); `products`
is the stability contract, `detail` may gain keys without notice. Data
updates (new harvest year, new weekly packs) change numbers under the
same paths.
