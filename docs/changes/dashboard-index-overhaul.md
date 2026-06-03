# Dashboard & Index Overhaul (June 2026)

This page explains, in plain terms, a set of changes made to the CPI engine and
dashboards: **what was broken, what we changed, where it lives, and how to run
it.** It is written for a mixed audience — statisticians and developers.

## The problem we set out to fix

On the dashboard, the four coloured division cards (Education, Transport,
Health, Insurance) and the headline National CPI were showing **0 or obviously
wrong numbers** — for example a national CPI of `30` when the real figure was
around `136`.

Two separate causes:

1. **Missing data was being counted as zero.** Each index is a *weighted
   average* of its parts. When a month had not been fully collected yet, the
   not‑yet‑computed parts had a value of `0` but still carried their full
   weight. Averaging real numbers together with those `0`s dragged the whole
   index down toward zero. So an incomplete month looked like prices had
   crashed, when really the data just was not in yet.

2. **The dashboard was reading a single tiny slice instead of the whole.** A
   "division" figure is really made of ~10 regional/segment sub‑rows (Kampala,
   Gulu, Arua, …) that must be combined by weight. The old dashboard grabbed
   *one arbitrary* sub‑row (often a region worth less than 2% of the division)
   and showed it as if it were the whole division.

## What we changed (and where)

All changes were made on the shared **`18.0`** base branch and then propagated
to each country (Uganda first, then Kenya, Tanzania, Zanzibar).

### 1. Only average the data that actually exists — *the real fix*
**Where:** `hcpi_index/models/computations/` — `national_index_update.py`,
`basket_coicop_index_update.py`, `basket_index_update.py`.

The averaging code used to say "include this part if it has a weight." It now
says "include this part if it has a weight **and** an actual computed value."
Parts that have not been computed yet (value `0`) are simply left out of the
average instead of dragging it down. An incomplete month is now an honest
average of what *has* been collected — no more collapse to zero.

### 2. One correct number per division per month
**Where:** new model `hcpi.national.division.index`
(`hcpi_index/models/national_indices/hcpi_national_division_index.py`),
computed during the National Index update.

Instead of the dashboard guessing which sub‑row to show, the engine now stores
**one properly weighted figure per division per month** (the weighted average
across all regions/segments). The dashboard just reads that one correct row.

### 3. Provisional index — a useful number before collection finishes
**Where:** `hcpi.national.index.update` — `refresh_provisional_division_index()`
and a **"Refresh Provisional"** button on the National Index Update form.

For the **in‑progress month** (the one still being collected), the system can now
produce a *provisional* estimate: for any item not yet priced this month, it
**carries forward that item's last known price**, then runs the normal
calculation. This gives a publishable estimate from day one of the month, which
firms up as real prices arrive.

- Provisional figures are stored **separately** (flagged `provisional`) and
  **never overwrite** the official series.
- Alongside it we show **"% of expected data collected"** so you know how
  complete the estimate is.

### 4. Redesigned analytical dashboard
**Where:** `hcpi_dashboard/` — `models/hcpi_dashboard.py`, `static/src/js/hcpi_dashboard.js`,
`static/src/xml/hcpi_dashboard.xml`.

- The **National CPI is now the first KPI card**, followed by three division
  cards (same style as before).
- The **historical trend chart** sits at half‑width just below the cards, next
  to the by‑division table.
- A **Month dropdown** at the top lets you view the figures as of any month.
- Empty/zero handling was hardened: the dashboard no longer silently blanks
  everything to zero when one piece fails — each section is guarded and errors
  are logged, and a real `0` is distinguished from "missing".

### 5. New "Operations" screen (collection monitoring)
**Where:** `hcpi_dashboard` — `operations_hooks()` in the model, plus
`static/src/js/hcpi_operations.js` and `static/src/xml/hcpi_operations.xml`; new
menu **Dashboard → Operations**.

For a selected month it shows: **% data collected** (overall and per division),
the **provisional CPI**, number of **collections**, **temporarily‑missing
items**, items **missing for more than 2 months**, and **outliers / inliers /
imputed** counts (read from the existing data‑validation records). It also has
the same **Month dropdown** and a **Refresh provisional** button.

## How it is deployed

The work lives on the `18.0` base worktree and is fanned out with
`bin/hcpi propagate`. Per the team workflow we **test Uganda first**, then the
other countries. A one‑time index **recompute** is needed after deploying so the
new per‑division table and corrected values are populated:

```bash
# in the base worktree, after committing on 18.0
bin/hcpi propagate            # merge 18.0 into all country branches
bin/hcpi update ug hcpi_index,hcpi_dashboard   # upgrade modules (creates the new table)
# then run, in the Odoo shell, the index updates in order:
#   Basket COICOP -> Basket -> National
# the National run also fills hcpi.national.division.index.
# finally press "Refresh Provisional" (or call refresh_provisional_division_index()).
```

To refresh the provisional in‑progress month at any time, press **Refresh
Provisional** on the National Index Update form (or the button on the Operations
screen).

## Things to keep in mind

- **Incomplete months are now shown honestly**, not hidden. The latest month's
  official figure is a weighted average of whatever has been collected so far;
  the **provisional** figure next to it is the carry‑forward estimate. Use the
  **% collected** indicator to judge how final a month is.
- **Provisional ≠ official.** Provisional rows are flagged and kept out of the
  published series; they are recomputed on demand.
- **Uganda's earlier bespoke fix was replaced.** Uganda's branch previously had
  its own version of this fix (a different weighting plus extra trace logging).
  To keep all four countries on one method, Uganda now uses the same base‑branch
  approach (which also adds a safeguard at the lowest level that the old Uganda
  version lacked). The old trace logging was dropped and can be re‑added if
  needed.
- The Operations screen reports the **in‑progress index month** by default; use
  the month dropdown to inspect other months. Outlier/inlier counts come from
  the data‑validation step, so they only appear for months that have been
  validated.
- The "items missing > 2 months" list is capped at 500 entries (shown as
  "500+").

## Still outstanding (nice‑to‑haves)

- A scheduled job to refresh the provisional month automatically (currently
  button‑driven).
- Re‑adding Uganda's detailed `_national_trace` diagnostics on the shared
  method, if the team still wants them.
