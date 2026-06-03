# HCPI Index Engine & Dashboard Overhaul (June 2026)

Plain-language summary of a set of changes to the CPI engine and dashboards:
**what was broken, what changed and where, how to operate it, and the things to
keep in mind.** Written for both statisticians and developers.

## The problems we set out to fix

1. **Division cards and the National CPI showed wrong or zero values.** For a
   month that had not been fully collected yet, the headline could read `0` or a
   nonsensical number (e.g. a national CPI of `30` when it should be ~`136`).
2. **No way to see a current-month figure before collection finished**, nor how
   much had been collected.
3. **No operational view** of collection status, missing items or data-quality
   counts.

## Why the numbers were wrong (root causes)

- **Missing data counted as zero.** Each index is a *weighted average* of its
  parts. Parts not yet computed had value `0` but still carried their full
  weight, so averaging real numbers with those `0`s dragged the whole index
  toward zero. An incomplete month therefore looked like prices had crashed.
- **The dashboard read one tiny slice instead of the whole.** A "division"
  figure is built from ~10 regional/segment sub-rows that must be combined by
  weight. The old dashboard showed one arbitrary sub-row (sometimes a region
  worth under 2%) as if it were the whole division.

## What changed (and where)

All changes live on the shared **`18.0`** base branch and are fanned out to each
country (`bin/hcpi propagate`). **Uganda is tested first**, then Kenya, Tanzania
and Zanzibar.

### 1. Only average data that exists — the real engine fix
`hcpi_index/models/computations/` — `national_index_update.py`,
`basket_coicop_index_update.py`, `basket_index_update.py`.

Every weighted-average step now includes a part only if it has **both a weight
and an actual computed value**. Not-yet-computed parts (value `0`) are left out
instead of dragging the average down. Incomplete months are now an honest
average of what has been collected.

### 2. One correct number per division per month
New model **`hcpi.national.division.index`**
(`hcpi_index/models/national_indices/hcpi_national_division_index.py`), computed
during the National Index run. It stores one properly weighted figure per
division per month (the weighted mean across all regions/segments). The
dashboard reads this single correct row.

### 3. Provisional index — a usable figure before collection finishes
`hcpi.national.index.update` — `refresh_provisional_division_index()` plus a
**"Refresh Provisional"** button on the National Index Update form.

For the **in-progress month**, the system carries forward each not-yet-priced
item's **last known price**, then runs the normal calculation, to produce a
*provisional* estimate. Provisional figures are stored **separately** (flagged)
and **never overwrite** the official series. A **collection-coverage** metric
("% of expected price cells collected") is computed alongside.

### 4. Redesigned analytical dashboard
`hcpi_dashboard/` — `models/hcpi_dashboard.py`, `static/src/js/hcpi_dashboard.js`,
`static/src/xml/hcpi_dashboard.xml`.

- A **Month selector** (top-right) — view the figures as of any month.
- **National CPI is the first KPI card**, then division cards. Each card shows a
  **month-over-month change pill** (▲/▼ %) and a sparkline. Division names are
  title-cased and truncate (full name on hover); all cards are equal height.
- A **historical trend** chart (half width) beside a **by-division table**.
- **Provisional tag logic:** the latest (in-progress) month is shown with a
  **"Provisional · X% collected"** tag, and its headline number is the official
  partial figure if one exists, otherwise the carry-forward estimate. Once a
  month is **both computed and fully collected**, it becomes a normal official
  month with no tag. Months with no computed index are hidden from the trend,
  the dropdown and the min/max — but the in-progress month stays selectable.
- Each dashboard section is guarded individually and errors are logged, so one
  failure can no longer blank the whole page to zero.

### 5. New "Operations" screen
`hcpi_dashboard` — `operations_hooks()` plus `static/src/js/hcpi_operations.js`
and `static/src/xml/hcpi_operations.xml`; menu **Dashboard → Operations**.

For a selected month: **% data collected** (overall and per division), the
**provisional CPI**, number of **collections**, **temporarily-missing items**
(with a count of those **missing > 2 months**), **outlier / inlier / imputed**
counts, and a month selector. Its month list spans observation, validation and
collection months so quality counts are always reachable (see note below).

## How to compute a month officially

The official index is built **bottom-up**, and the **Elementary-Aggregate step
is the foundation** — it must be run for new months before anything above it can
be computed. In the Index **Computations** menus, run in order:

1. **Update EA Indices** — `hcpi.ea.index.update.action_update_ea_indices`
2. **Update Basket COICOP Indices** — `action_update_basket_coicop_indices`
3. **Update Basket Indices** — `action_update_basket_indices`
4. **Update National Indices** — `action_update_national_indices`
   (also fills `hcpi.national.division.index`)

If a month's prices are collected but its official index still reads as
provisional/zero, it usually means the **EA step has not been run** for that
month yet (running only the higher levels reuses stale EA indices). Re-run from
step 1.

After deploying the code, do this once per database (Uganda first), then press
**Refresh Provisional** for the in-progress month.

## Deploy sequence (multi-country)

```bash
bin/hcpi propagate                         # merge 18.0 into all country branches
bin/hcpi update ug hcpi_index,hcpi_dashboard   # upgrade (creates the new table)
# recompute: EA -> Basket COICOP -> Basket -> National (see above)
# repeat for ke / tz / zar after Uganda passes
```

## Things to keep in mind

- **Incomplete months are shown honestly, clearly tagged provisional** — not
  hidden and not collapsed to zero. Use the **% collected** indicator to judge
  how final a month is.
- **Provisional ≠ official.** Provisional rows are flagged, kept out of the
  published series, and recomputed on demand.
- **Where outlier/inlier/imputed counts come from:** the data-validation step
  (`hcpi.data.validation`, TURKEY algorithm). They are bucketed by the
  **validation's month**, which can differ from the month the prices were
  observed (collection and observation dates are not always aligned in the
  data). If a month shows `0` outliers, select the month the validation was
  recorded under.
- The "items missing > 2 months" list is capped at 500 rows in the table, but
  the headline count is exact.

## Outstanding / nice-to-have

- A scheduled job to refresh the provisional month automatically (currently
  button-driven).
- Reconcile collection vs observation dating so quality counts and price months
  line up without manual month-switching.
