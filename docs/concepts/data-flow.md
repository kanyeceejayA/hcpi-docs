# Data Flow Walkthrough

This page traces one price observation from the moment an enumerator records it through to the headline CPI number on the dashboard. It's the **map** that ties together the [Module Reference](../understanding-the-codebase/module-reference.md), [CPI Concepts](cpi-concepts.md), and the [Country Variants](../understanding-the-codebase/country-variants.md) — read them first if you haven't.

By the end you should be able to answer: *"When I add a new outlet and an enumerator records a price, what does HCPI actually do with that data?"*

## The scenario

Concrete example we'll trace end-to-end:

> Enumerator Mary visits **Kikuubo Wholesale Market, Stall 4** on Tuesday morning, 16 September 2025. The questionnaire she opens on her phone lists 47 items priced at this outlet. For each, she records the price she observed. One of her entries: **white rice, 1 kg, retail — UGX 4,200**.
>
> Two days later, Supervisor Joseph validates Mary's questionnaire. Tukey flags two outlier prices (one shop sold rice at UGX 42,000 — clearly a typo). Joseph rejects those. Mary's UGX 4,200 entry passes.
>
> By the end of the month, the September national CPI is computed and shows on the dashboard. Our observation is one of millions that fed it.

We'll follow that data top-to-bottom.

## Phase 1: Setup (before the month starts)

Before any enumerator records anything, three things have to be in place:

### 1.1 The taxonomy

Defined once, rarely changed:

```
hcpi.division       "01 — Food and non-alcoholic beverages"
   └─ hcpi.group       "01.1 — Food"
       └─ hcpi.class       "01.1.1 — Bread and cereals"
           └─ hcpi.sub.class       "01.1.1.1 — Rice"
               └─ hcpi.micro.class       "01.1.1.1.1 — Rice (white)"
                   └─ hcpi.elementary.aggregate       "01.1.1.1.1.01 — Rice, white"
                       └─ hcpi.item       "white rice, 1 kg, retail"
```

All of this lives in `hcpi_coicop` (the hierarchy) and `hcpi_item` (the leaf items). Composite weights propagate from the EA upward.

### 1.2 The outlet and its catalog

For Kikuubo Stall 4:

```python
outlet = self.env['hcpi.outlet'].create({
    'name': 'Kikuubo Wholesale Market, Stall 4',
    'outlet_code': '255.53.21.03.004',
    'outlet_type_id': retail_type.id,
    'consumption_segment_id': urban_kampala_basket.id,
    'parish_id': nakasero_parish.id,        # ug_outlet overlay
})
```

The country overlay (`ug_outlet`) fills in `parish_id`; `sub_county_id`/`county_id`/`district_id`/`region_id` cascade automatically as stored-related fields.

Then, items the outlet sells are added through the **"Add Items" wizard** on the outlet form. Each creates one `hcpi.outlet.item`:

```python
self.env['hcpi.outlet.item'].create({
    'outlet_id': outlet.id,
    'item_id': white_rice_1kg.id,
    'base_price': 4000.00,    # set during the base period
    # ...
})
```

That `hcpi.outlet.item` is the **plan**: "we'll observe the price of white rice at this stall, every month, forever."

### 1.3 The questionnaire

At the start of September, the supervisor (or a scheduled job) creates one `hcpi.data.collection` per (outlet × month):

```python
collection = self.env['hcpi.data.collection'].create({
    'outlet_id': kikuubo_stall_4.id,
    'collection_date': fields.Date.from_string('2025-09-16'),
    'data_supervisor_id': joseph.id,
    'data_collector_id': mary.id,
    'state': 'draft',
})
collection.update_product_lines()  # populates collection_line from outlet's outlet_item_ids
```

`update_product_lines()` walks the outlet's active items and creates one `hcpi.data.collection.line` per item — Mary's checklist for the field visit.

## Phase 2: Field collection

State: `draft → survey`

Mary's mobile app fetches the questionnaire and shows her 47 lines. She walks the stall, prices each item, and types observations.

For our line:

```python
self.env['hcpi.data.observation.line'].create({
    'collection_line_id': rice_line.id,
    'observed_price': 4200.00,
    'observed_quantity': 1.0,
    'uoo_id': kg.id,
    'observation_date': fields.Datetime.now(),
})
```

One `hcpi.data.collection.line` typically has **multiple** `hcpi.data.observation.line` records — different vendors at the same stall, or repeat measurements. The line's `standard_price` (computed) averages them.

Mary submits. The collection state moves to `survey` and progress climbs as more lines get prices or codes.

If she found something unusual:

- **Out of stock?** She picks a `price_collection_code_ids` entry like "out of stock" instead of entering a price. The line is excluded from index computation cleanly, without being flagged as a zero-price problem.
- **Stock available but unusable price** (vendor refused, seasonal)? Different code, different downstream handling.

## Phase 3: Standardization

State: `survey → standardization`

If Mary recorded the price as "1 kg = UGX 4,200", no adjustment is needed — that matches the item's `standard_quantity`. But if she recorded "350 g sachet, UGX 1,500", standardization converts to per-kg:

```
unit_price = (1500 / 350) × 1000 = 4285.71 UGX per kg
```

This step exists because Mary doesn't always have an exact-1-kg portion to observe; she has to record what's actually on the shelf and let the system normalize.

The standardized prices live on the same `observation_line` records — the model has both `observed_price` (raw) and a computed standardized form used downstream.

## Phase 4: Validation

State: `standardization → validation`

This is where Joseph (the supervisor) comes in, and where the Tukey algorithm runs.

### 4.1 Tukey flagging

For each (item, basket, month) — say, "white rice, Kampala-urban basket, September 2025" — the system collects every observation:

```
Stall 1:  4,150
Stall 2:  4,200    ← our observation, Mary's
Stall 3:  4,250
Stall 4:  4,100
Stall 5: 42,000    ← typo, 10× too high
Stall 6:  4,300
Stall 7:  4,180
Stall 8:  4,220
Median = 4,210
Q1 = 4,150, Q3 = 4,275
IQR = 125
Threshold (k=3) = [4,150 − 3×125, 4,275 + 3×125] = [3,775, 4,650]
```

Stall 5's UGX 42,000 is way outside. Its collection line gets `is_outlier=True`. Everyone else is `is_inlier=True`.

(The computation lives in `hcpi.data.collection.line` and updates the booleans automatically when observations are saved.)

### 4.2 Supervisor review

Joseph opens his validation queue, sees the flagged outlier from Stall 5, confirms it's a typo (he can compare to short-term price relative — UGX 42,000 vs. last month's UGX 4,100 is a 10× jump), and rejects it: `is_rejected = True`.

He could equally accept a real outlier if, say, a supply shock genuinely changed the price for that vendor. The point of supervisor review is exactly the "is this anomaly real?" judgment that an algorithm can't make.

### 4.3 Zero-price checks

The `hcpi_computation` mixin runs its checks: are there enough non-zero observations? Is there at least a 6-month safe-history buffer? If yes, the questionnaire is cleared for indexing.

State: `validation → done`.

## Phase 5: Index computation

Once questionnaires are done, indices roll up. This happens **per-month**, triggered from the index wizards in `hcpi_index/`. Each wizard dispatches the actual computation as a background job through `queue_job`.

### 5.1 EA-level — geometric mean of price relatives

For our EA "white rice", September 2025, Kampala-urban basket:

```
Observations (inlier only):
  Stall 1:  4,150  →  price relative = 4150/4000 = 1.0375
  Stall 2:  4,200  →  1.0500   ← Mary's
  Stall 3:  4,250  →  1.0625
  Stall 4:  4,100  →  1.0250
  Stall 6:  4,300  →  1.0750
  Stall 7:  4,180  →  1.0450
  Stall 8:  4,220  →  1.0550

Geometric mean = (1.0375 × 1.0500 × 1.0625 × 1.0250 × 1.0750 × 1.0450 × 1.0550)^(1/7)
              = 1.0498

EA-level index for Kampala-urban basket, white rice, Sep 2025
  = 1.0498 × 100 = 104.98
```

Stored as one row in `hcpi.basket.elementary.aggregate.index`:

| field | value |
|---|---|
| `consumption_segment_id` | (Kampala-urban basket) |
| `elementary_aggregate_id` | (white rice EA) |
| `index` | 104.98 |
| `weight` | 0.0014 (from EA composite weight) |
| `date` | 2025-09-30 |
| `month` | 2025-09 |

### 5.2 Up the COICOP chain

The EA index gets aggregated up the COICOP hierarchy, weighted by composite weights:

```
basket.elementary.aggregate.index  (one row per EA × basket × month)
   ↓  geometric mean within micro-class, weighted by EA composite_weight
basket.micro.class.index           (one row per micro-class × basket × month)
   ↓  geometric mean within sub-class, weighted by micro-class composite_weight
basket.sub.class.index
   ↓  ...
basket.class.index
   ↓
basket.group.index
   ↓
basket.division.index
   ↓  weighted across divisions
basket.index                       (one number per basket per month)
```

Each level is its own model in `hcpi_index/`. Each level has a corresponding wizard (`ea.index.update`, `basket.coicop.index.update`, `basket.index.update`) that triggers its computation.

### 5.3 Across baskets — the national index

The final step combines all basket indices, weighted by population:

```
hcpi.basket.index (Kampala-urban):   1.0617
hcpi.basket.index (Kampala-low):     1.0584
hcpi.basket.index (rural-Uganda):    1.0512
... (other baskets)

weighted average by population share = 1.0557

hcpi.national.index for Sep 2025 = 105.57
```

That single number — September CPI is 105.57, meaning prices are 5.57% higher than the base period — is the **headline output** of the entire system.

In the database, it's one row in `hcpi.national.index`:

| field | value |
|---|---|
| `coicop` | (top — division aggregation) |
| `index` | 105.57 |
| `weight` | 1.0 |
| `date` | 2025-09-30 |
| `month` | 2025-09 |

## Phase 6: Display

The `hcpi.dashboard` model's `dashboard_hooks()` method queries:

- The last 12 months of division-level indices (filtered to `dashboard_display=True`)
- The last 24 months of national EA-level indices
- The list of baskets and items

That JSON returns to the OWL JS controller (`hcpi_dashboard.js`), which renders the line charts via ApexCharts. September 2025's 105.57 appears as the latest data point on the headline chart.

## What got written where

To recap, one observation involved writes to these tables:

| Phase | Tables written |
|---|---|
| 2. Field | `hcpi_data_observation_line` (one row) |
| 2. Field | `hcpi_data_collection` (state field updated) |
| 4. Validation | `hcpi_data_collection_line` (`is_outlier`, `is_inlier`, `is_rejected` recomputed) |
| 4. Validation | `hcpi_data_collection` (state updated to `done`) |
| 5. EA index | `hcpi_basket_elementary_aggregate_index` |
| 5. Up COICOP | `hcpi_basket_micro_class_index`, `..._sub_class_index`, `..._class_index`, `..._group_index`, `..._division_index`, `hcpi_basket_index` |
| 5. National | `hcpi_national_index` |

And the read side, when the dashboard opens:

| Phase | Tables read |
|---|---|
| 6. Display | `hcpi_basket_division_index`, `hcpi_national_index`, `hcpi_consumption_segment`, `hcpi_item`, `hcpi_class` (for `dashboard_display` filter) |

## What goes wrong, and where

A few common failure points worth knowing:

| Symptom | Likely phase | Likely fix |
|---|---|---|
| Index lurches up or down in one direction | Phase 4 — outliers got through validation | Re-open the questionnaires, re-check the Tukey flagging, reject the real outliers |
| Some items missing from a basket's index | Phase 1 — items aren't on the outlet's catalog | Add the items via the "Add Items" wizard |
| EA shows index but division doesn't | Phase 5 — `dashboard_display` is False on the division | Toggle the flag on `hcpi.class` |
| Index computation refuses to run | Phase 4 — zero-price validation failed | Check `zero_price_warning` field; ensure 6-month safe buffer |
| Background job stuck or not running | Phase 5 — `queue_job` worker not running | Check `server_wide_modules` has `queue_job`; check `workers > 0` in `hcpi.conf` |
| Dashboard charts blank | Phase 6 — view/JS failed | Check browser console; regenerate assets; check `hcpi_dashboard.js` is loading |

## Country variations of this flow

The phases above are identical across countries. What differs:

- **Phase 1.2 (outlet)**: the geographic FK depends on the country (Uganda: `parish_id`; Kenya: `zone_id`; Tanzania: `district_id`; Zanzibar: `collection_center_id`). See [Country Variants](../understanding-the-codebase/country-variants.md).
- **Phase 1.3 (questionnaire setup)**: KE, TZ, and Zanzibar have `xx_data_collection` modules adding **bulk questionnaire wizards** and **assignment wizards** — for batch-creating questionnaires across many outlets at once.
- **Phase 4 (validation)**: in countries with custom workflows, the state machine may have extra states. Uganda uses the core flow as-is.

## Where to read next

- **[CPI Concepts](cpi-concepts.md)** — re-read sections on price relatives and the Tukey algorithm with the worked example in mind.
- **[Module Reference](../understanding-the-codebase/module-reference.md)** — for each model mentioned above, see its full field list and place in the module hierarchy.
- **Day 10 of training** (coming soon) — hands-on with the validation step and the Tukey computation.
- **Day 3 of training** (coming soon) — hands-on with the ORM patterns used in `update_product_lines()`, `dashboard_hooks()`, and the index wizards.
