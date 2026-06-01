# Outlets

`hcpi_outlet` is **the hub** of HCPI. Almost every other module depends on it, directly or transitively. The collection workflow writes through it; the index computation reads through it; the dashboard charts roll up from observations stored under it.

If you only learn one module's internals, learn this one.

## What it owns

**Depends on:** `hcpi_item`, `hcpi_computation`

Three layers of model:

```
hcpi.outlet ───────► hcpi.outlet.type        (lookup: market / shop / supermarket / …)
   │
   ├─────────────► hcpi.consumption.segment  (the "basket" — urban / rural / …)
   │
   ├─────────────► hcpi.outlet.contact       (people at the outlet)
   │
   ├─────────────► hcpi.domestication        (demographic / geographic segment link)
   │
   └─►  hcpi.outlet.item                     (junction: item available at outlet)
              │
              └─► hcpi.outlet.item.observation  (time-series prices)
```

### `hcpi.outlet`

The collection point itself.

| Field | Purpose |
|---|---|
| `name` | Display name. |
| `outlet_code` | Validated against a strict format like `255.53.21.03.003`. The validation is in the model — see below. |
| `outlet_type_id` | FK to `hcpi.outlet.type`. |
| `consumption_segment_id` | FK to the basket — drives index computation downstream. |
| `latitude` / `longitude` | GPS, used in maps and the mobile app's "nearby" feature. |
| `contact_ids` | One2many to `hcpi.outlet.contact`. |
| `outlet_item_ids` | One2many to `hcpi.outlet.item` — the items priced at this outlet. |

Inherits `mail.thread` and `image.mixin` (so you can attach a photo).

### `hcpi.outlet.item` — the junction model

This is the **single most important model in HCPI**. One row per (outlet, item) pair — i.e., "Item X is priced at Outlet Y." Every observation hangs off one of these.

| Field | Purpose |
|---|---|
| `outlet_id` | FK to outlet. |
| `item_id` | FK to item. |
| `base_price` | Reference price for relative computations. |
| `observation_no` | Sequence within the (outlet, item). |
| `full_code` | Computed: combines outlet and item codes. |
| `consumption_segment_id` | Stored related from the outlet — duplicated here so basket queries don't have to join. |
| `elementary_aggregate_id` | Stored related from the item — same reason. |
| `observation_line` | One2many to `hcpi.outlet.item.observation`. |

**Performance:** There's a database index on `(elementary_aggregate_id, consumption_segment_id, id)` that makes basket-level lookups fast. At production scale (millions of rows) the index is what keeps queries sub-second.

### `hcpi.outlet.item.observation`

One row per time-series price.

| Field | Purpose |
|---|---|
| `outlet_item_id` | FK back to the junction. |
| `observation_date` | When the price was collected. |
| `observed_price` | The actual number. |
| `month` | Computed from `observation_date`. |
| `consumption_segment_id` | Stored related — same as above. |
| Short- and long-term price relatives | Computed via the `hcpi_computation` mixin (see [Data Collection](data-collection.md)). |

**Performance:** A DB index on `(outlet_item_id, observation_date DESC)` accelerates "give me the most recent N prices for this junction" — the shape every index computation needs.

### Other models

- **`hcpi.outlet.type`** — categorises outlets (retail, wholesale, market…). Tiny lookup.
- **`hcpi.consumption.segment`** — the basket. Groups outlets by consumer profile (urban/rural, income band). Has `outlet_ids` one2many. Driving entity for index roll-ups.
- **`hcpi.outlet.contact`** — name, phone, email per outlet.
- **`hcpi.domestication`** — links outlets to broader demographic/geographic segments.

## The outlet-code format

`outlet_code` is validated against a fixed shape — typically `255.53.21.03.003` (country.region.district.sub-region.outlet). The validation lives on `hcpi.outlet` itself. In country overlays (`ug_outlet`), the `parish_id` link reflects the same hierarchy. See [Country Overlay](country-overlay.md).

If you need a different format (different country, different number of levels), this is one place you'll change. The validation is a regex compare — search for `outlet_code` in `hcpi_outlet/models/hcpi_outlet.py`.

## Security

Two groups defined here:

- `group_outlet_user` — read on outlets, observations, junctions. Default for anyone who needs to see prices.
- `group_outlet_manager` — full CRUD. Restrict to data managers — adding or retiring outlets affects every downstream index.

For finer control (e.g. "supervisors only see outlets in their region"), real-world deployments add `ir.rule` records — see the [Country Overlay](country-overlay.md) page for examples.

## Where to look to change something

| You want to… | Open |
|---|---|
| Add a field to outlets | `hcpi_outlet/models/hcpi_outlet.py` + matching view in `hcpi_outlet/views/hcpi_outlet_views.xml` |
| Change the outlet-code format | `hcpi.outlet` — search for `outlet_code` and the regex/validation method |
| Add a new outlet type | Create an `hcpi.outlet.type` record via the UI, or seed it in `hcpi_outlet/data/` |
| Define a new basket | Create an `hcpi.consumption.segment` record. Then re-run index wizards. |
| Restrict who can see outlets | `hcpi_outlet/security/` — add an `ir.rule` record |
| Speed up a slow basket query | Confirm the existing indexes are being used (`EXPLAIN`). Add a new one in the model's `_sql_constraints` / migration file if needed. |
| Add a "contact role" (manager vs. clerk) | `hcpi.outlet.contact` — add a `role` Selection field |
| Change how `full_code` is composed on the junction | `hcpi.outlet.item` — find the `full_code` compute |
| Add a domestication tier | `hcpi.domestication` — add fields or related lookup table |
| Track changes to a specific outlet field | Add `tracking=True` on the field, then make sure `mail.thread` is inherited (it already is) |

## Common gotchas

- **`consumption_segment_id` is duplicated.** It exists on `hcpi.outlet`, on `hcpi.outlet.item`, and on `hcpi.outlet.item.observation` — as a stored related field on the junction and the observation. If you ever change how an outlet maps to a segment, **make sure the stored values get re-synced**. The compute should handle it on `write`, but if you bulk-update through SQL you'll need to refresh by hand.
- **Don't delete an outlet that has observations.** The cascade will take all the time-series with it. Prefer setting `active = False` (archiving) — it removes the outlet from default searches but keeps the history.
- **The `image.mixin`** means each outlet can have a photo. Stored as a `Binary(attachment=True)` so it sits in `ir.attachment`, not in the table. Backups need to include the filestore on disk too.

## Next

- **[Data Collection](data-collection.md)** — how observations get *written* to the models on this page.
- **[Indices & Dashboard](indices-dashboard.md)** — how observations get *read* into the CPI.
- **[Country Overlay](country-overlay.md)** — how `ug_outlet` extends this module to link outlets into Uganda's geography.
