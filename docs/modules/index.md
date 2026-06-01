# Module Deep Dives

The [Module Reference](../understanding-the-codebase/module-reference.md) is the bird's-eye view — every module on one page, what it owns, what it depends on. These pages are the **ground-level** view: pick the module you're touching, read its page, and you'll know which file to open and where to read more.

Each page follows the same shape:

- **What it does** — purpose in one paragraph.
- **The models** — every model worth knowing, with field-level detail.
- **State machines / important methods** — the workflows hidden inside the Python.
- **Where to look to change X** — a table of common modifications mapped to the file you'd open.

## How these pages are organised

```
Reference data    →    Data flow         →    Computation       →    Presentation
(taxonomy)             (collection)            (CPI math)             (UI on top)
─────────────         ─────────────           ──────────────         ──────────────
COICOP, Items,    →   Outlets,          →    Computation       →   Dashboard
Brands                Data Collection         mixin, Indices        + branding

                                                                    Country overlay
                                                                    (ug_*) wraps
                                                                    the data-flow
                                                                    side.
```

The pages:

1. **[Reference Data](reference-data.md)** — `hcpi_coicop`, `hcpi_item`, `hcpi_brand`. The taxonomy and item catalogue.
2. **[Outlets](outlets.md)** — `hcpi_outlet`. The hub: collection points, consumption segments, outlet-items, observations.
3. **[Data Collection](data-collection.md)** — `hcpi_data_collection`, `hcpi_data_collection_mobile_app`, plus the `hcpi_computation` mixin. The survey workflow and state machine.
4. **[Indices & Dashboard](indices-dashboard.md)** — `hcpi_index`, `hcpi_dashboard`. Where the CPI is computed and charted.
5. **[Country Overlays](country-overlay.md)** — Uganda, Kenya, Tanzania, Zanzibar. The geography overlays each EAC country ships, what makes them different, and the recipe for adding a new country.
6. **[Infrastructure & Third-party](infrastructure.md)** — `queue_job`, `web_progress`, `kola_web_enterprise`, `base_import_inherit`. Vendored modules HCPI relies on but didn't write.

## "I want to change X — where do I start?"

Quick jumps for common requests. Each row links to the deep-dive page that covers the change in detail.

| You want to change… | Start here |
|---|---|
| The COICOP hierarchy or weights | [Reference Data](reference-data.md) |
| Add a new item field (e.g. `barcode`) | [Reference Data](reference-data.md) |
| Branding, email templates, login screen | [Reference Data](reference-data.md) (`hcpi_brand`) + [Infrastructure](infrastructure.md) (`kola_web_enterprise`) |
| Outlet code format / validation | [Outlets](outlets.md) |
| Add a field to outlets | [Outlets](outlets.md) |
| Performance of basket queries | [Outlets](outlets.md) (indexes) |
| The survey state machine | [Data Collection](data-collection.md) |
| Add a "price collection code" (e.g. "out of stock") | [Data Collection](data-collection.md) |
| The mobile-app API surface | [Data Collection](data-collection.md) |
| The 6-month zero-price buffer | [Data Collection](data-collection.md) (`hcpi_computation`) |
| Add a new index level | [Indices & Dashboard](indices-dashboard.md) |
| A new special-index category | [Indices & Dashboard](indices-dashboard.md) |
| Dashboard chart range or series | [Indices & Dashboard](indices-dashboard.md) |
| Add a geographic level for a country | [Country Overlays](country-overlay.md) |
| Reassign a Kenyan zone's supervisor | [Country Overlays](country-overlay.md) (Kenya section) |
| Add bulk-questionnaire creation for a country | [Country Overlays](country-overlay.md) (TZ/ZAR wizards) |
| Add support for a brand-new country | [Country Overlays](country-overlay.md) (the recipe) |
| Tune index-compute parallelism | [Infrastructure](infrastructure.md) (`queue_job`) |
| Progress-bar colour | [Infrastructure](infrastructure.md) (`progress_bar_customization`) |

## A reading order if you're new

If you're touching HCPI back-end code for the first time, work top-to-bottom:

1. **[Reference Data](reference-data.md)** first — everything else references items and COICOP codes.
2. **[Outlets](outlets.md)** next — the hub almost every other module depends on.
3. **[Data Collection](data-collection.md)** — the workflow that produces the observations.
4. **[Indices & Dashboard](indices-dashboard.md)** — what happens to those observations.
5. **[Country Overlay](country-overlay.md)** + **[Infrastructure](infrastructure.md)** only when relevant.

You don't need to read everything cover-to-cover. The "Where to look" tables on each page are the part to bookmark.
