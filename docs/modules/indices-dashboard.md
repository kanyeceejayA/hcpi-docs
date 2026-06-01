# Indices & Dashboard

Two modules at the top of the data pyramid. `hcpi_index` is where the **CPI is computed**. `hcpi_dashboard` is where it gets **charted**.

If you came to HCPI thinking "I want to change how the index is calculated," `hcpi_index` is the page.

## `hcpi_index` — the CPI itself

**Depends on:** `hcpi_outlet`, `hcpi_coicop`, `web_progress`, `queue_job`

The biggest and most consequential module. Has four families of model and a stack of wizards.

### Model family 1 — national & domestic indices

Top-of-pyramid roll-ups, one per (COICOP code, month).

| Model | Purpose |
|---|---|
| `hcpi.national.index` | One index per (COICOP level, month) at the national level. Fields: `coicop` (the code as a string), `index` (the value), `weight`, `date`, computed `month`/`readable_date`. Unique on `(coicop, month)`. |
| `hcpi.domestic.index` | Same shape as national but for **domestic markets only**. Used when foreign / imported flows are stripped out. |

These are what published CPI numbers ultimately come from.

### Model family 2 — basket-level indices

One model per COICOP level, each scoped by `consumption_segment_id` (the basket). This is the **mid-tier** roll-up — before you can compute a national number for "food", you need the basket-level food index for urban, the one for rural, the one for high-income, etc.

| Model | Granularity |
|---|---|
| `hcpi.basket.index` | One per (basket, month) — the basket's top-level. |
| `hcpi.basket.division.index` | One per (basket, division, month). |
| `hcpi.basket.group.index` | One per (basket, group, month). |
| `hcpi.basket.class.index` | One per (basket, class, month). |
| `hcpi.basket.sub.class.index` | One per (basket, sub-class, month). |
| `hcpi.basket.micro.class.index` | One per (basket, micro-class, month). |
| `hcpi.basket.elementary.aggregate.index` | One per (basket, EA, month) — the most granular. |

A new index level (above EA, below division — anything in between) means adding a model in this family. The shape is identical: FK to basket, FK to COICOP level, `month`, `index`, `weight`, computes for display strings.

### Model family 3 — special indices

"Core CPI," "Food inflation," "Energy" — custom groupings that don't fit the COICOP hierarchy directly.

| Model | Purpose |
|---|---|
| `hcpi.special.index.category` | The category itself — name, code, which COICOP nodes it includes. |
| `hcpi.basket.special.index` | Per-basket value of the category. |
| `hcpi.national.special.index` | National roll-up of the category. |

Adding a new special index = create an `hcpi.special.index.category` record (via UI or data file) → re-run the special-index wizard → the per-basket and national rows materialise.

### Model family 4 — classification indices

Same three-tier pattern as special, for additional custom groupings (regional indices, age-band indices, anything that doesn't fit "special").

`hcpi.classification.index.category`, `hcpi.basket.classification.index`, `hcpi.national.classification.index`.

### The wizards

Transient models — open as forms, kick off work, then disappear. Each one dispatches the actual computation as a **background job** through `queue_job` (see [Infrastructure](infrastructure.md)), with progress reported through `web_progress`.

| Wizard | What it computes |
|---|---|
| `ea.index.update` | Elementary-aggregate-level basket indices (the most granular). |
| `basket.coicop.index.update` | Basket × COICOP-level indices (sub-class, class, group, division). |
| `basket.index.update` | Basket top-level index. |
| `national.index.update` | National roll-up across baskets. |
| `domestic.index.update` | Domestic-only national roll-up. |
| `special.index.update` | All special-category indices. |
| `classification.index.update` | All classification-category indices. |
| `hcpi.index.export.wizard` | Export indices to file (CSV/Excel). |

The recompute order matters: EA → Basket × COICOP → Basket top → National. Each step reads from the level below.

### `hcpi.class` inheritance — the dashboard flag

This module extends `hcpi.class` (from `hcpi_coicop`) with one extra field:

```python
dashboard_display = fields.Boolean()
```

That flag is what the dashboard reads to decide which classes to chart. Toggling it for a class makes the dashboard either show or hide that chart series.

### Security

Two groups:

- `index_user` — read on all index models. Default for analysts and the dashboard.
- `index_manager` — full CRUD + can run the wizards. Restrict carefully — re-running an index against the wrong month overwrites published numbers.

### Where to look to change something

| You want to… | Open |
|---|---|
| Add a new index level (between EA and division) | New basket-level model in `hcpi_index/models/` + matching wizard |
| Add a new special-index category | UI or `hcpi_index/data/` — create an `hcpi.special.index.category` record |
| Add a new classification-index category | Same shape, but using the classification models |
| Change index computation formulas | Search for `index = ` and `weight = ` in the relevant family's model file |
| Change which COICOP classes show on the dashboard | Toggle `dashboard_display` on `hcpi.class` (UI or shell) |
| Change how the export wizard formats | `hcpi.index.export.wizard` model + its method that builds the file |
| Tune parallelism of compute jobs | `hcpi.conf` `[queue_job]` channel definitions — see [Infrastructure](infrastructure.md) |
| Re-run an index after fixing a bug | Open the relevant wizard, pick the month, run. The job is async — watch progress in the progress bar. |

### Common gotchas

- **Order of recomputation matters.** EA → Basket × COICOP → Basket top → National. Skipping a level or running in the wrong order produces stale upper-tier numbers. The wizards encode the right order; if you call computation methods directly from the shell, replicate that order.
- **Stored compute fields on indices.** `month` / `readable_date` are stored computes. If you change how `month` is derived, you need to **rewrite** every existing row (a write loop with `mapped`) — Odoo doesn't backfill stored computes automatically.
- **The unique constraint on `(coicop, month)`.** Trying to compute twice for the same `(code, month)` raises. The wizards delete-then-recreate by design.

## `hcpi_dashboard` — the visual layer

**Depends on:** `hcpi_data_collection`, `hcpi_index`

**Purpose:** Renders the time-series and category charts shown on the HCPI landing page using **ApexCharts**.

### The thin model

There's only one model: `hcpi.dashboard`. It exposes a single method, `dashboard_hooks()`, which returns a JSON blob:

- Division-level indices for the **last 12 months** (filtered by `dashboard_display=True`).
- National EA-level indices for the **last 24 months**.
- The list of baskets.
- The list of items.
- The year range to render across the chart x-axis.

The frontend JS (`hcpi_dashboard.js`) calls this method, gets the JSON, and builds the chart series.

### The assets bundle

| File | Purpose |
|---|---|
| `hcpi_dashboard/static/lib/apexcharts.js` (or CDN) | The ApexCharts library. |
| `hcpi_dashboard/static/src/css/custom-13.css` | Layout, colours, sizes. |
| `hcpi_dashboard/static/src/js/hcpi_dashboard.js` | The OWL component that calls `dashboard_hooks()` and renders the charts. |
| `hcpi_dashboard/static/src/xml/hcpi_dashboard.xml` | The QWeb template for the OWL component. |

All bundled via the manifest's `assets` key.

### Where to look to change something

| You want to… | Open |
|---|---|
| Add a new chart series | `hcpi_dashboard/models/hcpi_dashboard.py` — extend `dashboard_hooks()` to return the extra series, then add a chart definition in `hcpi_dashboard.js` |
| Change the date window (12/24 months) | `dashboard_hooks()` — search for `relativedelta` / `months=` |
| Toggle which divisions are charted | Set `dashboard_display` on the `hcpi.class` records — no code change needed |
| Change chart colours or types | `hcpi_dashboard.js` — the ApexCharts series options |
| Re-style the layout | `hcpi_dashboard/static/src/css/custom-13.css` and the QWeb template |
| Add a click-through (chart → detail) | `hcpi_dashboard.js` — add an `events: { dataPointSelection: ... }` handler that does `this.env.services.action.doAction(...)` |
| Investigate why a chart is empty | Check `dashboard_display` on the COICOP classes; check that the relevant index rows exist for the displayed months |

### Common gotchas

- **Empty dashboard means empty data, not broken code.** The dashboard renders whatever `dashboard_hooks()` returns. If a chart is blank, the underlying index hasn't been computed for the displayed months. Run the wizard.
- **ApexCharts is loaded lazily.** If you add a chart and nothing renders, check the browser console — a missing import in `hcpi_dashboard.js` is the usual cause.
- **OWL re-renders on every navigation.** Don't put expensive computation in the component lifecycle — keep it in `dashboard_hooks()` on the server side where queries can be cached.

## How indices & dashboard connect

```
Observations (hcpi_outlet)
        │
        ▼
[ EA-level indices ]     ←─ wizard: ea.index.update
        │
        ▼
[ Basket × COICOP-level indices ]   ←─ wizard: basket.coicop.index.update
        │
        ▼
[ Basket top-level + National indices ]   ←─ wizards: basket.index.update, national.index.update
        │
        │       (plus parallel "special" and "classification" tracks)
        │
        ▼
hcpi.dashboard.dashboard_hooks()
        │
        ▼  (JSON)
hcpi_dashboard.js   ←  ApexCharts renders
```

When you "change how the dashboard looks," you're usually editing the bottom of this chain. When you "change how the CPI is calculated," you're at the top. Different files, different mental model — but the pipeline above ties them together.

## Next

- **[Data Collection](data-collection.md)** — what feeds the EA-level indices.
- **[Outlets](outlets.md)** — the master data the basket roll-ups segment by.
- **[Infrastructure](infrastructure.md)** — `queue_job` and `web_progress`, which the wizards depend on.
