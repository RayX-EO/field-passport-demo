# Field Passport — demo site (static)

Self-contained static site, ~9 MB, no build step, all internal paths relative:
- `index.html`: the farmer demo. 8 constructed farms (`farms/`), each 15 real neighbouring winter-wheat fields;
  the grouping into farms is ours, and every farm says so. Each field's report was computed in advance.
- `examples/`: 6 single example fields in the same format. The page offers them only when there are no farms.
- `field_passport_onepager.*` (and `_a4`, the portrait version for Word): the proposal one-pager, drawn from
  FARM-05's reports.

The site calls no API, so a visitor cannot submit a field of their own. The live service and its contract are in
yield-platform `services/passport` (`CONTRACT.md`, `RUNBOOK.md`).

## How it is served

GitHub Pages, from the root of `main`, at **passport.rayx.co.uk** (`CNAME`; DNS `CNAME passport -> <owner>.github.io`
at Namecheap; Enforce HTTPS on). It is not linked from rayx.co.uk. Merging into `main` publishes.

## The reports

Since 2026-10-15 they come from the 13-season record, 2013–2025 (Landsat 2013–2017 resampled to 10 m, Sentinel-2
2018–2025), computed by the engine the live service runs. Fields where fewer than 3 wheat seasons are Sentinel-2
carry the reminder that their within-field pattern is less reliable, and the page shows it. Before, the reports came
from the 8-season Sentinel-2 record (2018–2025), built on 2026-08-28.

To recompute them, for example after a new release of the record, refresh a checkout of this repository on JASMIN,
from a checkout of yield-platform (`services/passport/demo/refresh_demo.py`):

```bash
SITE=~/field-passport-demo sbatch services/passport/jasmin/demo.sbatch
```

It keeps every field, its outline and its id, and rewrites the reports, the farms' summaries, `farms/index.json`,
`examples/index.json` and the list embedded in `index.html` (`window.PASSPORT_CFG`), then redraws the one-pagers
(`proposal_onepager.py`). Review the diff, then merge.

## Data

The field outlines are UKCEH LCCM 2022 parcels. The reports carry no crop history.
