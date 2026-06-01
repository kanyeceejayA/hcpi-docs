# Reference Data — COICOP, Items, Brands

Three modules together define the **taxonomy** every observation gets classified under, the **catalogue** of priced items, and the **cosmetic** layer (branding, emails). Nothing in HCPI works without these — every collection line, every outlet item, every index is ultimately a record hanging off something defined here.

## `hcpi_coicop` — the classification spine

**Depends on:** `mail`

**Purpose:** Implements the international **COICOP** (Classification of Individual Consumption by Purpose) hierarchy — a 6-level tree from broad categories ("Food and non-alcoholic beverages") down to the smallest aggregation unit before individual items.

### The six levels

Each level is its own model. They form a parent-child chain via FKs.

| Model (`_name`) | Level | Example |
|---|---|---|
| `hcpi.division` | 1 (top) | `01` Food and non-alcoholic beverages |
| `hcpi.group` | 2 | `01.1` Food |
| `hcpi.class` | 3 | `01.1.1` Bread and cereals |
| `hcpi.sub.class` | 4 | `01.1.1.1` Rice |
| `hcpi.micro.class` | 5 | `01.1.1.1.01` Long-grain rice |
| `hcpi.elementary.aggregate` | 6 (bottom) | `01.1.1.1.01.001` Imported long-grain rice — the parent of actual items |

Each level has:

- `code` — the level-local code (`01`, `1`, `01`, …).
- `composite_weight` — the aggregation weight, used when rolling indices up.
- A `name` and `description`.
- A FK to its parent level.

The elementary aggregate computes a **`full_code`** by concatenating the chain: `01.01.01.01.01.001`. That string is what shows up on outlet items, observations, and basket rows everywhere downstream.

All six models inherit `mail.thread` so structural changes (renaming, re-weighting) appear in the chatter for audit.

### Security

Two groups, defined in `hcpi_coicop/security/`:

- `coicop_user` — read-only on every model. The default for analysts who consume the taxonomy.
- `coicop_manager` — full CRUD. Restrict to a small team — changes here ripple through every CPI run.

### Where to look to change something

| You want to… | Open |
|---|---|
| Add or rename a division | `hcpi_coicop/models/hcpi_division.py` + data file under `hcpi_coicop/data/` if seeded |
| Change a level's weight | The level's model file (e.g. `hcpi.class.py`) — look at `composite_weight` |
| Change how `full_code` is composed | `hcpi.elementary.aggregate` — search for `full_code` compute |
| Restrict who can edit the taxonomy | `hcpi_coicop/security/ir.model.access.csv` and the group definitions next to it |
| Migrate to a new COICOP revision | New data files seeding the new codes, plus migration scripts to move existing items |

## `hcpi_item` — the catalogue of priced things

**Depends on:** `hcpi_coicop`

**Purpose:** Defines the actual items whose prices get collected ("1 kg sugar", "500 ml cooking oil") and the units they're priced in. Every observation in the database eventually traces back to an `hcpi.item` row.

### The models

**`hcpi.item`** — the catalogue entry. Key fields:

| Field | Purpose |
|---|---|
| `name` | Display name. |
| `code` | Auto-incremented within its elementary aggregate when not supplied. |
| `elementary_aggregate_id` | FK to the EA — that's how the item slots into the COICOP tree. |
| `uoo_id` | Unit of observation (`hcpi.uoo`). |
| `standard_quantity` | The quantity at which prices are typically collected (e.g. "1" for a kg). |
| `is_standard` | If `True`, the item is pre-portioned — not re-weighed at the outlet. |
| `full_code` | Computed: `<ea_code>.<code>`. The reference string everywhere. |

Inherits `mail.thread` for change tracking. Validates duplicate names per EA on both `create()` and during import — see the validation methods in `hcpi.item`.

**`hcpi.uoo`** — unit of observation (pieces, kg, litres, dozen, …). Tiny model; just a `name` and a `code`.

### Where to look to change something

| You want to… | Open |
|---|---|
| Add a new field to items (e.g. `barcode`) | `hcpi_item/models/hcpi_item.py` + matching view in `hcpi_item/views/` |
| Change the duplicate-name validation | `hcpi.item` — look for the `_check_*` methods and the `@api.constrains` |
| Add a new unit of measure | Create a `hcpi.uoo` record via the UI or seed it in `hcpi_item/data/` |
| Mark items as "standard" in bulk | Use the ORM via the [Odoo shell](../training/module/part1-models.md#step-8-meet-the-orm-via-the-odoo-shell) — `env['hcpi.item'].search([('elementary_aggregate_id.code', '=', 'X')]).write({'is_standard': True})` |
| Change how the item code increments | `hcpi.item` — find the create() override that derives `code` from the EA |

## `hcpi_brand` — branding, emails, login

**Depends on:** `auth_signup`, `mail`

**Purpose:** The **cosmetic** layer. No business logic, no models of its own — just the visual customisation HCPI ships out of the box.

### What's inside

- **`static/src/menu.js`** — custom menu tweaks (logo, app launcher behaviour).
- **`static/src/custom.css`** — colour overrides, sizing, layout tweaks for the navbar and login pages.
- **`data/` email templates** — signup, password reset, account activation. These are `mail.template` records that the auth_signup flow renders.
- **`security/`** — rules tied to the email templates (who can edit them).

### `hcpi_brand` vs `kola_web_enterprise`

Both modules customise the UI. The difference:

- **`hcpi_brand`** — country-agnostic HCPI defaults: logo, login screen, transactional emails.
- **`kola_web_enterprise`** — replaces deeper parts of the web UI (navbar layout, SCSS variables). See [Infrastructure](infrastructure.md).

When you change look-and-feel, you'll usually touch one or both.

### Where to look to change something

| You want to… | Open |
|---|---|
| Change the logo | `hcpi_brand/static/description/icon.png` for the app tile; the login template under `hcpi_brand/views/` for the login page |
| Change a transactional email | `hcpi_brand/data/` — the `mail.template` records |
| Tweak the navbar | `hcpi_brand/static/src/custom.css` + the OWL/QWeb templates under the same folder |
| Replace the navbar layout entirely | `kola_web_enterprise` — see [Infrastructure](infrastructure.md) |
| Add a new "welcome" email | New `<record model="mail.template">` in `hcpi_brand/data/`, then register it from wherever you want it sent (likely auth_signup) |

## How these three connect

```
COICOP tree                     Item catalogue           Branding (separate)
hcpi.division                                            login screen
   ↓                                                     emails
hcpi.group                                               custom navbar
   ↓
hcpi.class                      hcpi.uoo (units)
   ↓                              ↑
hcpi.sub.class                  hcpi.item ──────► used everywhere downstream:
   ↓                              ↑                outlet items, observations,
hcpi.micro.class                  │                collection lines, indices
   ↓                              │
hcpi.elementary.aggregate ────────┘
```

If you're adding a new priced-product category, your edits ripple in this order:

1. Add the COICOP path down to a new elementary aggregate (`hcpi_coicop`).
2. Add the item that lives under that EA (`hcpi_item`).
3. Make sure outlets that should price it are linked through `hcpi.outlet.item` ([Outlets](outlets.md)).

## Next

- **[Outlets](outlets.md)** — where items get linked to collection points.
- **[Data Collection](data-collection.md)** — where prices for those items are recorded.
