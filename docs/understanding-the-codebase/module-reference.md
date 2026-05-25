# Module Reference

This page walks through every module in `custom/HCPI/` — what it owns, what depends on it, and when you'd touch it. It's the page to come back to when you see a model name in a stack trace and need to figure out which folder it lives in.

[Understanding the Codebase](index.md) covers what an Odoo module *is* and what's inside one. This page covers what each specific module *does*.

## Why so many modules?

HCPI ships as 17 module folders. That feels like a lot for one product. The reason is Odoo's modular philosophy: each coherent piece of functionality lives in its own module, and modules declare their dependencies on others. Splitting things up gives you three things:

1. **A country can swap one piece without touching the rest.** Uganda's geography lives in `ug_location/`. Kenya would replace it with `ke_location/` and leave everything else alone.
2. **Loading is incremental.** When you install or update only one module, only its files are re-read — meaning faster dev iteration.
3. **The dependency graph documents itself.** `hcpi_dashboard` depends on `hcpi_index`, which depends on `hcpi_outlet`, which depends on `hcpi_item`, which depends on `hcpi_coicop`. Reading the `__manifest__.py` of any module tells you exactly what it needs to function.

## The four kinds of module in HCPI

Every HCPI module falls into one of four roles:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  REFERENCE DATA               COMPUTATION                       │
│  (the taxonomy)               (turns data into the CPI)         │
│  ┌─────────────┐              ┌─────────────────┐               │
│  │ hcpi_coicop │              │ hcpi_computation│               │
│  │ hcpi_item   │              │ hcpi_index      │               │
│  │ hcpi_brand  │              │ hcpi_dashboard  │               │
│  └─────────────┘              └─────────────────┘               │
│         │                              ▲                        │
│         ▼                              │                        │
│  DATA FLOW                             │                        │
│  (collects observations)               │                        │
│  ┌─────────────────────────────────────┴───┐                    │
│  │ hcpi_outlet                             │                    │
│  │ hcpi_data_collection                    │                    │
│  │ hcpi_data_collection_mobile_app         │                    │
│  └─────────────────────────────────────────┘                    │
│                                                                 │
│  COUNTRY CUSTOMIZATION (Uganda's overlay)                       │
│  ug_location · ug_outlet · ug_base_import                       │
│                                                                 │
│  INFRASTRUCTURE / THIRD-PARTY                                   │
│  queue_job · web_progress · kola_web_enterprise · ...           │
└─────────────────────────────────────────────────────────────────┘
```

The data flows top-to-bottom on the left side (taxonomy → outlets/items get assigned codes → collection records observations) and then up the right side (observations get aggregated into indices and shown on the dashboard).

## Dependency graph

The HCPI modules form a clean DAG. Reading bottom-up gives you the install order:

```
hcpi_dashboard ──┐
                 ├──▶ hcpi_index ──┐
                 │                 │
                 ├──▶ hcpi_data_collection ──┐
                 │                           │
                 │   ┌─ hcpi_data_collection_mobile_app
                 │   │
                 │   ├─ ug_outlet ────────▶ ug_location
                 │   │
                 │   └─ ug_base_import ──▶ base_import_inherit
                 ▼
            hcpi_outlet
                 │
                 ├──▶ hcpi_item ──▶ hcpi_coicop
                 │
                 └──▶ hcpi_computation
```

Three things to notice:

- **`hcpi_coicop` is the foundation.** Everything that classifies a price (which item? which division?) traces back here.
- **`hcpi_outlet` is the hub.** Both data collection (writing observations) and index computation (reading observations) sit on top of it.
- **`hcpi_computation` is a thin mixin.** It exposes one abstract model used by collection lines, observations, and every index variant.

---

## Reference data modules

These define the static taxonomy — the categories, codes, and labels that everything else hangs off.

### `hcpi_coicop`

**Purpose:** Defines the international **COICOP** (Classification of Individual Consumption by Purpose) hierarchy that HCPI uses to group items. Six levels deep, top to bottom.

**Depends on:** `mail`

**Key models** — each is a level in the hierarchy:

| Model (`_name`) | Level | What it represents |
|---|---|---|
| `hcpi.division` | 1 (top) | E.g. "01 Food and non-alcoholic beverages" — the 12-14 broad categories |
| `hcpi.group` | 2 | Sub-division (e.g. "01.1 Food") |
| `hcpi.class` | 3 | E.g. "01.1.1 Bread and cereals" |
| `hcpi.sub.class` | 4 | Sub-class |
| `hcpi.micro.class` | 5 | Micro-class |
| `hcpi.elementary.aggregate` | 6 (bottom) | The smallest aggregation unit — the parent of actual items |

Each level has a `code` and a `composite_weight`. The elementary aggregate computes a `full_code` like `01.01.01.01.01.01` by concatenating the chain. All models inherit `mail.thread` so changes show up in the chatter.

**When you'd edit it:** updating the COICOP structure (new revision), adding or renaming a division/class, changing how weights aggregate up the hierarchy.

**Security:** `coicop_user` (read-only) and `coicop_manager` (full CRUD) groups defined here.

### `hcpi_item`

**Purpose:** Defines the actual items whose prices are collected (e.g. "1 kg sugar", "500 ml cooking oil"). Each item hangs off an elementary aggregate.

**Depends on:** `hcpi_coicop`

**Key models:**

- **`hcpi.item`** — the catalog entry. Fields: `name`, `code` (auto-incremented within its EA if not provided), `elementary_aggregate_id` (FK to the EA), `uoo_id` (unit of observation), `standard_quantity`, `is_standard` (if true, the item isn't re-weighed during collection). Computes `full_code` as `<ea_code>.<code>`. Inherits `mail.thread`. Validates against duplicate names per EA on both `create` and import.
- **`hcpi.uoo`** — unit of observation (pieces, kg, litres, etc.).

**When you'd edit it:** maintaining the item master list, changing standard quantities, marking items as standard (already-portioned and so not re-weighed in the field).

### `hcpi_brand`

**Purpose:** Branding and email-template layer — the cosmetic side of the system. No business logic; no models of its own.

**Depends on:** `auth_signup`, `mail`

**What's in it:** custom menus (`menu.js`), CSS (`custom.css`), email templates for things like signup and password reset, and security rules tied to the templates.

**When you'd edit it:** customizing the look-and-feel for a new country, changing email wording, or restyling the navbar.

---

## Data flow modules

These are where observations are recorded and validated. The path is `outlet → questionnaire → collection lines → observations`.

### `hcpi_outlet`

**Purpose:** Defines the physical or logical points where prices are collected (markets, shops), the consumption-segment baskets they belong to, and the items they sell. This is the **hub** of the system — everything observation-related goes through here.

**Depends on:** `hcpi_item`, `hcpi_computation`

**Key models:**

| Model | What it represents |
|---|---|
| `hcpi.outlet` | A single collection point. Fields include `outlet_code` (validated against a strict format like `255.53.21.03.003`), `outlet_type_id`, `consumption_segment_id`, `latitude`/`longitude`, `contact_ids`, `outlet_item_ids`. Inherits `mail.thread` and `image.mixin`. |
| `hcpi.outlet.type` | Categorizes outlets (retail, wholesale, market...). |
| `hcpi.consumption.segment` | The "basket" — groups outlets by consumer profile (urban/rural, income band, etc.). Has `outlet_ids` one2many. |
| `hcpi.outlet.contact` | Contact info attached to an outlet (name, phone, email). |
| `hcpi.outlet.item` | The junction: an item available at an outlet, with `base_price`, `observation_no`, computed `full_code`, and `observation_line` one2many. A DB index on `(elementary_aggregate_id, consumption_segment_id, id)` accelerates basket-level lookups. |
| `hcpi.outlet.item.observation` | One time-series price observation: `observation_date`, `observed_price`, `month` (computed), `consumption_segment_id`, plus short-term and long-term price relatives. Has its own DB index on `(outlet_item_id, observation_date DESC)`. |
| `hcpi.domestication` | Links outlets to demographic/geographic segments. |

**When you'd edit it:** managing outlets, defining outlet types or consumption segments, changing how outlet codes are validated, indexing for performance.

**Performance note:** the explicit DB index on observations is what makes basket-level queries fast at scale — there can be millions of observation rows over time, so the index matters.

### `hcpi_data_collection`

**Purpose:** Implements the **survey workflow**. Every visit by an enumerator to an outlet produces one *questionnaire* (`hcpi.data.collection`) with one *line* per item priced.

**Depends on:** `hcpi_outlet`, `hcpi_computation`

**Key models:**

- **`hcpi.data.collection`** — a questionnaire. Fields: `outlet_id`, `collection_date`, `data_supervisor_id`/`data_collector_id` (FKs to `res.users`), `state` (state machine: `draft → survey → standardization → validation → done/cancel`), computed `progress` (%), `data_validation_id` (`ondelete='set null'` — detach on validation deletion), `draft_collection_line` / `collection_line` (one2many). The `update_product_lines()` method auto-creates collection lines from the outlet's active items.
- **`hcpi.data.collection.line`** — one item's observations within a questionnaire. Fields: `outlet_item_id`, `standard_price` (computed from observations), `price_collection_code_ids` (many2many — codes like "out of stock"), `observation_line` (one2many), `is_outlier`/`is_inlier` (computed booleans), `is_rejected`, price-relative fields. Inherits `hcpi.computation` so zero-price warnings work here.
- **`hcpi.data.observation.line`** — individual price entry (`observed_price`, `uoo_id`, `observed_quantity`).
- **`hcpi.price.collection.code`** — codes that explain non-normal observations (e.g. "out of stock", "seasonal unavailable"). Maintained as data.
- **`hcpi.data.validation`** — wraps a batch of collections for supervisor review.

**Security groups defined here:** `group_data_collection_supervisor`, `group_data_collection_collector`, `group_data_collection_statician`. Day 7 walks through these.

**When you'd edit it:** changing the survey workflow, the state machine, adding new "price collection codes", or tweaking validation rules.

### `hcpi_data_collection_mobile_app`

**Purpose:** Mobile-specific extensions of the collection models — what the Flutter app talks to.

**Depends on:** `hcpi_data_collection`

**Key models:** thin `_inherit` extensions of `hcpi.data.collection`, `hcpi.outlet`, `hcpi.outlet.contact`, `hcpi.item`, `res.config.settings`, and `res.users`. Most actual logic lives in the mobile client; the backend exposes endpoints and stores app config.

**When you'd edit it:** modifying mobile sync logic, offline data structures, mobile-app settings exposed through `res.config.settings`, or per-user mobile preferences.

**Day 12** covers the mobile API in detail.

---

## Computation and analysis modules

These take the observations the data-collection modules wrote and turn them into the CPI.

### `hcpi_computation`

**Purpose:** A thin **abstract mixin** — defines one model that other models inherit to get zero-price validation logic.

**Depends on:** `base`

**Key model:**

- **`hcpi.computation`** (`AbstractModel`) — exposes four fields (`zero_price_has_issues`, `zero_price_has_safe_months`, `zero_price_warning` text, `zero_price_warning_html`) and a `get_zero_price_validation_status()` method. The method checks for zero-price observations and requires a buffer of at least **6 safe months** with **at most 5 problematic items** before allowing index computation.

This mixin is inherited by `hcpi.data.collection.line`, `hcpi.outlet.item.observation`, and every concrete index model in `hcpi_index`. That's why it's not just a function — being a mixin lets each consumer carry its own validation state as fields.

**When you'd edit it:** adjusting the 6-month buffer, the 5-item limit, or the warning text.

### `hcpi_index`

**Purpose:** Where the CPI is actually computed. Defines indices at every level of the hierarchy — national, domestic, basket-level, classification-level, and special — and the wizards that trigger their computation.

**Depends on:** `hcpi_outlet`, `hcpi_coicop`, `web_progress`, `queue_job`

This is the biggest and most consequential module. Models break into four families:

**National & domestic indices** (top-of-pyramid):

| Model | Purpose |
|---|---|
| `hcpi.national.index` | One index per (COICOP level, month) at the national level. Fields: `coicop` (char identifying COICOP code), `index` (float), `weight`, `date`, computed `month`/`readable_date`. Unique on `(coicop, month)`. |
| `hcpi.domestic.index` | Same shape as national but for domestic markets only. |

**Basket-level indices** (per consumption segment, at every COICOP level):

| Model | Purpose |
|---|---|
| `hcpi.basket.index` | Top-level basket index — one per `consumption_segment_id` per month. |
| `hcpi.basket.division.index` | One per (basket, division, month). |
| `hcpi.basket.group.index` | One per (basket, group, month). |
| `hcpi.basket.class.index` | One per (basket, class, month). |
| `hcpi.basket.sub.class.index` | One per (basket, subclass, month). |
| `hcpi.basket.micro.class.index` | One per (basket, micro-class, month). |
| `hcpi.basket.elementary.aggregate.index` | One per (basket, EA, month). |

**Special indices** (custom categories like "food inflation", "core CPI"):

| Model | Purpose |
|---|---|
| `hcpi.special.index.category` | The category definition. |
| `hcpi.basket.special.index` | Per-basket roll-up of a special category. |
| `hcpi.national.special.index` | National roll-up of a special category. |

**Classification indices** (additional custom groupings, parallel to "special"):

`hcpi.classification.index.category`, `hcpi.basket.classification.index`, `hcpi.national.classification.index` — same three-tier pattern.

**Wizards** (transient models — they open as forms, trigger work, then disappear):

- `ea.index.update` — recompute EA-level indices
- `basket.coicop.index.update`
- `basket.index.update`
- `national.index.update`
- `domestic.index.update`
- `special.index.update`
- `classification.index.update`
- `hcpi.index.export.wizard` — export indices to file

Each wizard dispatches the actual computation as a **background job via `queue_job`**, with progress reported through `web_progress`. This is why `hcpi_index` depends on both of those infrastructure modules.

**`hcpi.class` inheritance:** this module extends `hcpi.class` (the COICOP class model) with a `dashboard_display` boolean. That flag is what the dashboard uses to pick which divisions to chart.

**Security:** `index_user` (read) and `index_manager` (CRUD).

**When you'd edit it:** the computation formulas, adding new index levels, defining new special or classification categories, changing how jobs are queued.

### `hcpi_dashboard`

**Purpose:** The visual layer — charts of national, division-level, and basket trends rendered in the browser with **ApexCharts**.

**Depends on:** `hcpi_data_collection`, `hcpi_index`

**Key model:**

- **`hcpi.dashboard`** — a thin pseudo-model exposing one method, `dashboard_hooks()`, that returns:
    - division-level indices for the last 12 months (filtered to `dashboard_display=true`),
    - national EA-level indices for the last 24 months,
    - list of baskets,
    - list of items,
    - the year range to render.

The method returns JSON; the front-end JS (`hcpi_dashboard.js`) consumes it and draws the charts.

**Assets bundled:** ApexCharts (CDN), `custom-13.css`, `hcpi_dashboard.js`, and the `hcpi_dashboard.xml` template.

**When you'd edit it:** rearranging the dashboard, changing which series are charted, adjusting time windows, tweaking colours or chart types, or adding new widgets.

---

## Country customization (the `ug_*` overlay)

A pattern worth understanding before you adapt HCPI for another country.

The generic `hcpi_*` modules are country-agnostic — they don't know what a "district" is. The `ug_*` modules layer Uganda-specific structure on top. When Kenya adopts HCPI, the work is: write a parallel `ke_*` set of overlay modules.

### `ug_location`

**Purpose:** Uganda's geographic hierarchy.

**Depends on:** `mail`, `hcpi_data_collection`

**Key models** (five-level hierarchy):

| Model | Level |
|---|---|
| `ug.region` | Top |
| `ug.district` | Within region |
| `ug.county` | Within district |
| `ug.sub.county` | Within county |
| `ug.parish` | Within sub-county (smallest unit) |

All inherit `mail.activity.mixin`. Codes are auto-formatted (zero-padded under 10).

**When you'd edit it:** updating Uganda's administrative geography (rare).

### `ug_outlet`

**Purpose:** Connects `hcpi.outlet` to `ug.parish` — so a Uganda outlet has a location, which other countries' overlays won't.

**Depends on:** `hcpi_outlet`, `ug_location`

**Key model:**

- **`hcpi.outlet` (inherited)** — adds `parish_id` (FK), plus *stored related* fields up the chain: `sub_county_id`, `county_id`, `district_id`, `region_id`. All cascade from `parish_id`. Storing them as related fields (rather than computing on read) makes filtered queries — "all outlets in Kampala" — efficient.

**When you'd edit it:** linking outlets to the location structure, modifying how location cascades.

**The pattern to copy for another country:** create a `ke_location/` with `ke.region/county/...`, then a `ke_outlet/` that adds the equivalent FKs to `hcpi.outlet`.

### `ug_base_import`

**Purpose:** Uganda-specific import validation (e.g., parsing Uganda location codes).

**Depends on:** `base_import_inherit`, `ug_location`, `hcpi_data_collection`

**Key model:**

- **`base_import.import` (inherited)** — same as the parent extension with Uganda-specific logic on top.

**When you'd edit it:** changing how Uganda CSV/Excel imports are validated.

---

## Import & data validation

### `base_import_inherit`

**Purpose:** A generic extension of Odoo's `base_import` that adds validation and batch-optimization for HCPI data.

**Depends on:** `base_import`

**Key model:**

- **`base_import.import`** (inherited transient) — adds a `use_optimized_import` boolean and pre-insertion validation that prevents duplicates.

**When you'd edit it:** tweaking CSV/Excel import validation, adding new bulk-import optimizations, or improving error messages.

---

## Infrastructure / third-party modules

These aren't HCPI's own code. They're vendored to keep installs reproducible (no hunting for the right OCA branch). Treat as read-only unless you have a specific reason to patch.

### `queue_job`
OCA's job queue framework. Lets long-running operations (index computation, bulk import) run asynchronously in background workers. `hcpi.conf` declares it in `server_wide_modules` because it has to load before any database is opened.

### `web_progress`
UI for long-running operations — adds a modal with progress percentage and a "run in background" button. Wired into the index-computation wizards.

### `kola_web_enterprise`
Branding/theming layer. Replaces parts of Odoo's web UI (navbar, layout, SCSS variables) with HCPI's look.

### `progress_bar_customization`
CSS override changing the progress bar color from yellow to orange. Cosmetic only.

---

## Quick reference: "where do I look to change X?"

| If you want to change... | Edit module |
|---|---|
| Add a new COICOP division | `hcpi_coicop` |
| Add an item (programmatically) | `hcpi_item` |
| Add a new outlet field | `hcpi_outlet` |
| Restrict who can see outlets | `hcpi_outlet/security/` |
| Add a new survey state (e.g. "review") | `hcpi_data_collection` |
| Change the "out of stock" code list | `hcpi_data_collection` (data file) |
| Add a new index level | `hcpi_index` |
| Add a new special-index category | `hcpi_index` (`hcpi.special.index.category`) |
| Tweak the dashboard chart range | `hcpi_dashboard` (`dashboard_hooks()`) |
| Add a new geographic level to Uganda | `ug_location` + `ug_outlet` |
| Adapt for a new country | New `xx_location/`, `xx_outlet/`, `xx_base_import/` |
| Change branding / login screen | `hcpi_brand` + `kola_web_enterprise` |
| Tune zero-price validation threshold | `hcpi_computation` |
| Tune index-computation parallelism | `hcpi.conf` `[queue_job]` section |

## Next steps

- **[Making Your First Edits](../first-edits/index.md)** — apply this map: pick a module and make a small, safe change to it.
- **Day 7 of training (coming soon)** — covers the security groups touched by `hcpi_coicop`, `hcpi_outlet`, `hcpi_data_collection`, and `hcpi_index`.
- **Day 8–11 of training (coming soon)** — deep dives into Coicop, Location, Outlet/Item, Data Collection, and Importation modules.
