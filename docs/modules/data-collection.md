# Data Collection

Three modules together: the **survey workflow**, the **mobile-app surface**, and the **zero-price validation mixin**. Together they cover everything between an enumerator visiting an outlet and an observation being stored.

## `hcpi_data_collection` — the survey workflow

**Depends on:** `hcpi_outlet`, `hcpi_computation`

**Purpose:** Implements the questionnaire-and-line workflow that turns "an enumerator visited an outlet" into structured price observations the index can read.

### The hierarchy of models

```
hcpi.data.collection           ← the questionnaire (one visit, one outlet)
       │
       ├── hcpi.data.collection.line          ← one priced item within the questionnaire
       │         │
       │         └── hcpi.data.observation.line  ← individual price entries
       │
       └── hcpi.data.validation              ← a batch wrapper for supervisor review
```

### `hcpi.data.collection` — the questionnaire

One row per visit. Key fields:

| Field | Purpose |
|---|---|
| `outlet_id` | Which outlet was visited. |
| `collection_date` | When. |
| `data_collector_id` | Enumerator (`res.users`). |
| `data_supervisor_id` | Supervisor (`res.users`). |
| `state` | State machine — `draft → survey → standardization → validation → done` (or `cancel`). |
| `progress` | Computed % — how much of the expected work is done. |
| `data_validation_id` | FK to a batch validation wrapper. `ondelete='set null'` — detaching the batch doesn't delete the questionnaire. |
| `draft_collection_line` | One2many while in draft. |
| `collection_line` | One2many for finalised lines. |

### The state machine

This is the heart of the module — five states in order:

```
  draft  ──►  survey  ──►  standardization  ──►  validation  ──►  done
    │                                                              ▲
    └───────────────────────────────────────────────────►  cancel  │
                                                                   │
                                  (re-opening a "done" → back to validation)
```

Each transition lives as a Python method on `hcpi.data.collection`:

- `action_start_survey()` — `draft → survey`
- `action_start_standardization()` — `survey → standardization`
- `action_submit_for_validation()` — `standardization → validation`
- `action_approve()` — `validation → done`
- `action_cancel()` — anywhere → `cancel`
- `action_reset_to_draft()` — `cancel → draft`

When you need to **change the workflow**, these methods are where you start.

### Auto-creating lines from the outlet

`hcpi.data.collection` has an **`update_product_lines()`** method that auto-creates one `hcpi.data.collection.line` per active item at the outlet. Called when a questionnaire transitions out of draft. If you add a new way to pick items (e.g., "only items priced last month"), this is the method to edit.

### `hcpi.data.collection.line` — one priced item

One row per item within the questionnaire.

| Field | Purpose |
|---|---|
| `outlet_item_id` | FK to the (outlet, item) junction. |
| `standard_price` | Computed from observations once collected. |
| `price_collection_code_ids` | Many2many — codes like "out of stock", "seasonal unavailable" that explain the missing/odd values. |
| `observation_line` | One2many to `hcpi.data.observation.line`. |
| `is_outlier` / `is_inlier` | Computed booleans flagged by validation algorithms. |
| `is_rejected` | Manual override. |
| Price-relative fields | Computed via the `hcpi_computation` mixin (see below). |

Inherits `hcpi.computation` for zero-price warnings.

### `hcpi.data.observation.line` — the actual price entry

One row per **price actually written** — multiple per line are common (an enumerator records 3 spot prices for the same item).

| Field | Purpose |
|---|---|
| `observed_price` | The number. |
| `uoo_id` | Unit of observation (matches the item's UoO). |
| `observed_quantity` | How much was priced. |

### `hcpi.price.collection.code` — the exception codes

The list of "why is this missing/odd" labels — `OUT_OF_STOCK`, `SEASONAL`, `NEW_PRODUCT`, etc. Maintained as data records.

To add a new code (e.g. `STRIKE`):

1. Add a `<record model="hcpi.price.collection.code">` in `hcpi_data_collection/data/`.
2. Restart Odoo with `-u hcpi_data_collection`.
3. Users see it in the dropdown immediately.

### `hcpi.data.validation` — batch supervisor review

Wraps a group of questionnaires for cross-cutting review (e.g. "all of Kampala's collections from this month"). Lighter than a Collection — mostly a state and a list of children.

### Security

Three groups defined here, with `implied_ids` building a triangle:

```
group_data_collection_collector       (file own questionnaires)
       implied by
group_data_collection_supervisor      (review any collector's work)
       implied by
group_data_collection_statician       (run validation algorithms, edit codes)
```

A user in `statician` has all three roles' permissions; a user in `supervisor` has supervisor + collector. Same pattern you built in [Module Part 3](../training/module/part3-security.md#how-real-hcpi-modules-set-this-up).

### Where to look to change something

| You want to… | Open |
|---|---|
| Add a state to the workflow | `hcpi.data.collection` — the `state` Selection + add an `action_*` method + button on the form view |
| Change which items get auto-loaded as lines | `update_product_lines()` on `hcpi.data.collection` |
| Add a new "price collection code" | `hcpi_data_collection/data/` — add a `<record model="hcpi.price.collection.code">` |
| Add a field to the questionnaire form | `hcpi.data.collection` model + `hcpi_data_collection/views/` |
| Change how outliers are flagged | The `is_outlier` / `is_inlier` computes on `hcpi.data.collection.line` |
| Restrict who can advance state | `_require_supervisor()`-style methods in the model + `has_group` checks |
| Change the supervisor review batch | `hcpi.data.validation` model and its views |
| Tweak `tracking=True` for audit | Add `tracking=True` to fields on `hcpi.data.collection` (it already inherits `mail.thread`) |

## `hcpi_data_collection_mobile_app` — the Flutter surface

**Depends on:** `hcpi_data_collection`

**Purpose:** The thin server-side companion to the Flutter mobile app — where most of the actual collection work happens.

### What's inside

Mostly `_inherit` extensions of existing models:

- `hcpi.data.collection` — adds mobile-specific fields (sync state, offline markers).
- `hcpi.outlet` — exposes app-specific lookups.
- `hcpi.outlet.contact` — same.
- `hcpi.item` — adds mobile-display tweaks.
- `res.config.settings` — global app settings exposed in the HCPI settings page.
- `res.users` — per-user mobile preferences.

### How the Flutter app talks to this

Not via custom controllers. The mobile app calls the standard **XML-RPC endpoint** at `/xmlrpc/2/object`, exactly as covered in [Part 3 of the module tutorial](../training/module/part3-security.md#controllers-and-the-rpc-endpoint-that-replaces-them). The mobile app reads, creates, and writes records on the same models the web UI uses — `hcpi.data.collection`, `hcpi.data.observation.line`, etc.

What this module adds is the **fields and methods** the mobile app expects to find. If the Flutter app calls `env['hcpi.data.collection'].submit_offline_batch(...)`, the method definition is here.

### Where to look to change something

| You want to… | Open |
|---|---|
| Add a field synced from the mobile app | `hcpi_data_collection_mobile_app/models/hcpi_data_collection.py` (the `_inherit` block) |
| Change what happens when mobile submits a batch | Look for `submit_*` methods on `hcpi.data.collection` here |
| Expose a new mobile config option | `res.config.settings` `_inherit` here, plus its view inheritance under `views/` |
| Override per-user mobile prefs | `res.users` `_inherit` here |
| Debug an API call from the app | Server logs — `tail -f log/hcpi.log` and look for `/xmlrpc/2/object` entries showing the method name and args |

## `hcpi_computation` — the zero-price mixin

**Depends on:** `base`

**Purpose:** A tiny abstract model — one mixin everything else inherits to get **zero-price validation**.

### What it provides

`hcpi.computation` is an `AbstractModel`. It exposes:

| Field / method | Purpose |
|---|---|
| `zero_price_has_issues` | Boolean — are there problematic zero-price observations? |
| `zero_price_has_safe_months` | Boolean — has enough time passed to safely ignore? |
| `zero_price_warning` | Text — human-readable summary. |
| `zero_price_warning_html` | Same, HTML formatted for forms. |
| `get_zero_price_validation_status()` | The method that computes the four above. |

### The hard-coded thresholds

The method checks two things:

- **At least 6 safe months** of buffer.
- **At most 5 problematic items** within that buffer.

If both pass, computation can proceed. If not, the warning surfaces in the UI and the wizard refuses to compute.

These two numbers are **the most-asked-to-tune values** in the whole codebase. They live in `hcpi_computation/models/hcpi_computation.py` — search for `6` and `5`, the comments tell you which is which.

### Who inherits this mixin

- `hcpi.data.collection.line` — every line carries its own zero-price status.
- `hcpi.outlet.item.observation` — every observation can be checked.
- Every concrete index model in [`hcpi_index`](indices-dashboard.md).

That's why it's a mixin and not a function: each consumer carries its own state as actual fields. Compute methods on those consumers all delegate to `get_zero_price_validation_status()`.

### Where to look to change something

| You want to… | Open |
|---|---|
| Change the 6-month buffer | `hcpi_computation/models/hcpi_computation.py` — `get_zero_price_validation_status()` |
| Change the 5-item limit | Same method |
| Change the warning wording | Same method — `zero_price_warning` and `zero_price_warning_html` |
| Add a new validation criterion | Add a field to the mixin, recompute it in the same method, and surface it on the inheriting models' forms |
| Remove zero-price validation entirely | Don't — it gates index computation for a reason. If you must, override `get_zero_price_validation_status()` on a specific subclass and return safe values. |

## How the three modules connect

```
                  ┌───────────────────────────────────────┐
                  │           hcpi_computation            │
                  │     (abstract zero-price mixin)       │
                  └───────────────────────────────────────┘
                                  ▲    ▲     ▲
                                  │    │     │
                            inherits  inherits  inherits
                                  │    │     │
┌────────────────────┐   ┌────────────────────────────┐   ┌──────────────────────────┐
│ hcpi_data_         │   │ hcpi_data_collection       │   │ hcpi_index               │
│ collection_mobile_ │◄──┤ (survey workflow + lines + │──►│ (indices that read       │
│ app                │   │  validation batches)       │   │  collection lines)       │
└────────────────────┘   └────────────────────────────┘   └──────────────────────────┘
```

When you add a new validation rule, you almost always touch `hcpi_computation` (the rule itself) and one or more of `hcpi_data_collection`, the mobile app inherit, and `hcpi_index` (the consumers).

## Next

- **[Indices & Dashboard](indices-dashboard.md)** — what happens to the validated observations.
- **[Outlets](outlets.md)** — the master data the questionnaires reference.
