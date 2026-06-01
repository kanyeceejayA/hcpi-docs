# Country Overlays — Uganda, Kenya, Tanzania, Zanzibar

Each EAC country that has adopted HCPI ships a small **overlay** of modules that layers its own administrative geography, outlet linkage, and (sometimes) workflow tweaks on top of the generic `hcpi_*` core. The generic modules don't know what a "district" is; the overlays fill those slots in.

This page covers all four current overlays — **Uganda**, **Kenya**, **Tanzania**, **Zanzibar** — and ends with the recipe for adding a fifth.

See [Country Variants](../understanding-the-codebase/country-variants.md) for the broader story of branches and how country code is kept separate from the EAC base.

## At a glance

| Country | Branch | Modules | Location levels | Notes |
|---|---|---|---|---|
| **Uganda** | `ug_18` | `ug_location`, `ug_outlet`, `ug_base_import` | 5 (region → district → county → sub-county → parish) | No own `data_collection` overlay — uses the base directly. |
| **Kenya** | `ke_18_mtnce` | `ke_location`, `ke_outlet`, `ke_data_collection`, `ke_base_import` | 3 (region → county → zone) | Zone is the assignment unit; supervisor/collector live on it. Adds a **`review`** state. |
| **Tanzania** | `tz_18_mtnce` | `tz_location`, `tz_outlet`, `tz_data_collection`, `tz_base_import` | 2 (region → district) | District is the assignment unit. Adds **bulk + assign questionnaire** wizards. |
| **Zanzibar** | `zar_18` | `zar_location`, `zar_outlet`, `zar_data_collection` | 3 (region → district → collection center) | Uses "collection center" as the leaf — closer to operational reality. No own `base_import`. |

The level count and the model that sits at the bottom — `parish` / `zone` / `district` / `collection center` — is the single biggest structural difference between countries. Everything else hangs off that choice.

## Uganda (`ug_18`)

The original adopter and the model the others follow most closely.

### `ug_location` — five-level hierarchy

```
ug.region ──► ug.district ──► ug.county ──► ug.sub.county ──► ug.parish (leaf)
```

Each level:

- `name` (`Char`, required, tracked).
- `code` — auto-padded with a leading zero under 10 (so `1` displays as `01`).
- FK to its parent.
- Inherits `mail.activity.mixin`.

The parish is the leaf — what an outlet attaches to.

### `ug_outlet` — extends `hcpi.outlet`

```python
class HcpiOutletInherit(models.Model):
    _inherit = 'hcpi.outlet'

    parish_id = fields.Many2one('ug.parish', ondelete='restrict')

    sub_county_id = fields.Many2one('ug.sub.county',
                                    related='parish_id.sub_county_id', store=True)
    county_id     = fields.Many2one('ug.county',
                                    related='sub_county_id.county_id', store=True)
    district_id   = fields.Many2one('ug.district',
                                    related='county_id.district_id', store=True, readonly=False)
    region_id     = fields.Many2one('ug.region',
                                    related='county_id.region_id', store=True)
```

The stored-related chain is the pattern every country uses for fast filtering — "all outlets in Kampala district" becomes a single equality, no joins required.

### `ug_base_import`

Tiny module. Plugs Uganda-specific validation into `base_import_inherit` (mostly Uganda location-code parsing).

### What's *missing* from the Uganda overlay

**No `ug_data_collection`.** Uganda uses the base `hcpi_data_collection` state machine and workflow unchanged. Supervisor/collector assignment happens by other means (manual on the questionnaire, or via the base groups).

## Kenya (`ke_18_mtnce`)

Three location levels and the assignment unit moved into the geography.

### `ke_location` — three-level hierarchy

```
ke.region ──► ke.county ──► ke.zone (leaf)
```

- `ke.region` and `ke.county` are standard — name, code, padded `region_code` / `county_code` computes, `mail.thread` + `mail.activity.mixin`.
- **`ke.zone` is where Kenya gets interesting.** It carries the supervisor and collector assignment directly:

```python
class KeZone(models.Model):
    _name = 'ke.zone'
    _inherit = ['mail.thread', 'mail.activity.mixin']

    name      = fields.Char(required=True, tracking=True)
    county_id = fields.Many2one('ke.county', required=True, tracking=True)
    region_id = fields.Many2one('ke.region',
                                related='county_id.region_id', store=True)
    code      = fields.Integer(required=True, tracking=True)

    # Assignment moved into the location hierarchy
    data_supervisor_id = fields.Many2one('res.users',
        domain=lambda self: [
            ('groups_id', 'in',     self.env.ref('hcpi_data_collection.group_data_collection_supervisor').id),
            ('groups_id', 'not in', self.env.ref('hcpi_data_collection.group_data_collection_statician').id),
        ])
    data_collector_id  = fields.Many2one('res.users', domain=lambda self: [...])

    data_supervisor_email = fields.Char()
    data_collector_email  = fields.Char()

    zone_code = fields.Char(compute='_compute_zone_code')  # "01.02.03"
```

That `domain=lambda self: [...]` on the M2O fields is **filtered by group membership** — only users in the `group_data_collection_supervisor` group (and not also in `_statician`) can be picked as a zone's supervisor. Same idea for the collector.

### `ke_outlet` — extends `hcpi.outlet`

```python
zone_id    = fields.Many2one('ke.zone', ondelete='restrict')
zone_code  = fields.Integer(related='zone_id.code', store=True, readonly=False)
county_id  = fields.Many2one('ke.county', related='zone_id.county_id', store=True)
region_id  = fields.Many2one('ke.region', related='zone_id.region_id', store=True)
```

Shorter chain than Uganda's because Kenya only has three levels.

### `ke_data_collection` — adds a `review` state

This is where Kenya diverges from the base workflow. Two things happen:

**1. The state machine gains a `review` step:**

```python
state = fields.Selection(selection=[
    ('draft',           'Draft'),
    ('survey',          'Price Survey'),
    ('review',          'Review'),           # ← Kenya-specific
    ('standardization', 'Standardization'),
    ('validation',      'Validation'),
    ('done',            'Done'),
    ('cancel',          'Cancelled'),
], default='draft', tracking=True)
```

**2. Supervisor and collector flow through from the zone, not set manually:**

```python
zone_id            = fields.Many2one('ke.zone',  related='outlet_id.zone_id')
data_supervisor_id = fields.Many2one('res.users', related='outlet_id.zone_id.data_supervisor_id')
data_collector_id  = fields.Many2one('res.users', related='outlet_id.zone_id.data_collector_id')
```

When you create a questionnaire for an outlet, the people responsible are already known — the zone has them.

**3. A `submit_for_review` and a customised `validate` method** that auto-creates an `hcpi.data.validation` record per (zone, month) so reviews are batched by area and time period.

### `ke_base_import`

Same shape as `ug_base_import` — Kenya-specific row validation on top of `base_import_inherit`.

## Tanzania (`tz_18_mtnce`)

Two location levels — the simplest hierarchy of the four.

### `tz_location` — two-level hierarchy

```
tz.region ──► tz.district (leaf)
```

- `tz.region` — standard region model with `code` validation (`@api.constrains('code')` rejects zero).
- `tz.district` — like Kenya's zone, **carries supervisor and collector assignment** with the same `groups_id`-filtered domain.

### `tz_outlet` — extends `hcpi.outlet`

```python
district_id   = fields.Many2one('tz.district', ondelete='restrict')
district_code = fields.Integer(related='district_id.code', store=True, readonly=False)
region_id     = fields.Many2one('tz.region',  related='district_id.region_id', store=True)
```

### `tz_data_collection` — adds bulk + assign wizards

Tanzania's workflow extension is wizard-driven rather than state-driven (no extra `review` state like Kenya).

| Wizard | What it does |
|---|---|
| `bulk_questionnaire_wizard` | Create many questionnaires at once across selected outlets/items. |
| `assign_questionnaire_wizard` | Reassign existing questionnaires to a different supervisor or collector. |

If your country needs **mass questionnaire creation** at the start of each pricing cycle, this is the pattern to copy.

### `tz_base_import`

Same shape as the others.

## Zanzibar (`zar_18`)

Three levels, with the leaf named for what it actually is.

### `zar_location` — three-level hierarchy

```
zar.region ──► zar.district ──► zar.collection.center (leaf)
```

The leaf is named **`zar.collection.center`** rather than a generic administrative term — closer to the operational reality (a collection centre is a specific physical place price data is gathered).

`zar.region` exposes smart-button counts and a `supervisor_id` at the regional level:

```python
class ZarRegion(models.Model):
    _name = 'zar.region'

    code = fields.Char(required=True, tracking=True)  # Note: Char, not Integer
    name = fields.Char(required=True, tracking=True)
    supervisor_id = fields.Many2one('res.users', string='Regional Supervisor')

    district_ids       = fields.One2many('zar.district', 'region_id')
    district_count     = fields.Integer(compute='_compute_district_count', store=True)
    collection_center_count = fields.Integer(compute='_compute_collection_center_count', store=True)
```

`zar.collection.center` carries the operational assignment and computes a hierarchical code:

```python
class ZarCollectionCenter(models.Model):
    _name = 'zar.collection.center'
    _inherit = ['mail.thread', 'mail.activity.mixin']

    name        = fields.Char(required=True, tracking=True)
    code        = fields.Char(required=True, tracking=True)
    district_id = fields.Many2one('zar.district', required=True, tracking=True)
    region_id   = fields.Many2one('zar.region',
                                  related='district_id.region_id', store=True, readonly=True)

    # "01.02" — region.code + center.code, joined
    collection_center_code = fields.Char(compute='_compute_collection_center_code', store=True)

    supervisor_id      = fields.Many2one('res.users',
                                         related='region_id.supervisor_id', readonly=False)
    data_collector_id  = fields.Many2one('res.users')
```

Notice **`supervisor_id` is related from the region** — Zanzibar manages supervision at the region level and inherits it down. The collector is set per centre.

### `zar_outlet` — extends `hcpi.outlet`

```python
collection_center_id = fields.Many2one('zar.collection.center', ondelete='restrict', tracking=True)
district_id          = fields.Many2one('zar.district',
                                       related='collection_center_id.district_id', store=True, readonly=True)
region_id            = fields.Many2one('zar.region',
                                       related='collection_center_id.region_id', store=True, readonly=True)
```

### `zar_data_collection` — same wizards as Tanzania

Bulk + assign questionnaire wizards, no state-machine modification.

### What's *missing* from the Zanzibar overlay

**No `zar_base_import`.** Zanzibar uses `base_import_inherit` directly. If you find yourself adding country-specific import validation for Zanzibar, this is the gap to fill.

## Common patterns across all four

Despite the differences in level count and assignment unit, every country overlay does the same five things:

| Pattern | Where you see it |
|---|---|
| **Inherit `mail.thread`/`mail.activity.mixin`** on location models | All location levels in all four countries. |
| **`code` zero-padding** via compute or formatting | UG (auto), KE (`_compute_*_code`), TZ (`_compute_*_code`), ZAR (string concat). |
| **Stored related FKs on `hcpi.outlet`** up the location chain | All four — the fast-filter pattern. |
| **`ondelete='restrict'`** on outlet → leaf-location FK | All four — protects outlets from accidental geography deletion. |
| **`group_data_collection_*`-filtered domains** on supervisor/collector pickers | KE (on zone), TZ (on district), ZAR (per region). UG does this on the base questionnaire instead. |

And two patterns specific to some countries:

| Pattern | Countries |
|---|---|
| **Assignment moved into the geography** (`data_supervisor_id` + `data_collector_id` on the leaf location, not the questionnaire) | Kenya (`ke.zone`), Tanzania (`tz.district`), Zanzibar (`zar.collection.center`). Uganda does not — it stays on the questionnaire. |
| **Bulk / assign wizards** | Tanzania, Zanzibar. Kenya and Uganda don't ship them. |
| **Extra workflow state (`review`)** | Kenya only. |

## The recipe for a new country

Standing up HCPI for a fifth country (say, **Rwanda** = `rw_*`):

### Step 1: branch

Cut a new branch off `18.0`:

```bash
git checkout 18.0
git checkout -b rw_18
```

Country code lives on this branch only. The generic `hcpi_*` modules on `18.0` don't change.

### Step 2: `rw_location/`

Mirror the country's actual administrative hierarchy. Don't copy Uganda's five levels just because — match what the national statistics office uses. Rwanda might be: `rw.province → rw.district → rw.sector → rw.cell` (4 levels).

Files:

```
rw_location/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   ├── rw_province.py
│   ├── rw_district.py
│   ├── rw_sector.py
│   └── rw_cell.py
├── data/
│   └── rw_geography.xml      ← seed the real hierarchy from NSO data
├── security/
│   ├── security.xml
│   └── ir.model.access.csv
└── views/
    ├── rw_province_views.xml
    ├── ...
    └── rw_location_menus.xml
```

Decide on each level whether to:

- Track changes (`mail.thread`).
- Carry supervisor/collector assignment (the KE/TZ/ZAR pattern) or leave that on the questionnaire (the UG pattern).

### Step 3: `rw_outlet/`

Mirror one of the existing outlet overlays. Pick the leaf location as the M2O target, then stored-related FK up the chain.

### Step 4 (optional): `rw_data_collection/`

Only if you need to:

- Add a workflow state (like KE's `review`).
- Add bulk/assign wizards (like TZ/ZAR).
- Override supervisor/collector flow-through from the geography.

If the base workflow does what you want, **skip this module** — Uganda does. Less code, fewer surfaces to maintain.

### Step 5 (optional): `rw_base_import/`

Only if you need country-specific import validation (parsing Rwanda location codes from CSV/Excel headers, etc.). Zanzibar skipped this — country with simple imports can use `base_import_inherit` directly.

### Step 6: update `addons_path`

`hcpi.conf` `addons_path` is what tells Odoo where to find the new modules. After dropping `rw_location/`, `rw_outlet/`, etc. into `custom/`, restart Odoo with `-u rw_location -u rw_outlet -u rw_data_collection -u rw_base_import` (or update all from the Apps screen with developer mode on).

## Where to look to change something

| You want to… | Open |
|---|---|
| Add a geographic level to Uganda | `ug_location/models/` — new model + FK on the level above + view file |
| Reassign a Kenyan zone's supervisor | `ke.zone` form (UI) — and the `data_supervisor_id` filter domain if you need to broaden who's pickable |
| Add a workflow state for a country | The country's `*_data_collection` (Kenya is the worked example); if there isn't one, create it |
| Add a bulk questionnaire wizard | Copy Tanzania's `bulk_questionnaire_wizard` model + view, adjust for the country's location leaf |
| Change Zanzibar's supervisor delegation | `zar.region.supervisor_id` cascades down to `zar.collection.center.supervisor_id` via `related=` — change the cascade or the source |
| Show the full location breadcrumb on the outlet form | The country's `*_outlet/views/hcpi_outlet_views.xml` — add the stored-related fields |
| Reject outlets with invalid location codes on import | The country's `*_base_import/models/base_import.py` (create the module if missing) |
| Change which users are pickable as data supervisors | The `domain=lambda self: [...]` on the leaf location's `data_supervisor_id` field |

## Common gotchas

- **`store=True` related fields can drift.** If you bulk-update via SQL or restore an old backup, the stored-related `region_id`/`district_id`/etc. on outlets may not match the current geography. Re-trigger the compute by writing the source field back to itself in the shell:
    ```python
    env['hcpi.outlet'].search([]).write({'parish_id': self.parish_id})  # for UG
    ```
- **Don't put country logic in the generic modules.** "Just one tiny `if country == 'KE'`" in `hcpi_outlet` is exactly how core gets polluted. Use `_inherit` from the overlay.
- **Match the country's real geography.** Don't copy Uganda's five-level model out of habit. Tanzania has two because that's what TZ uses.
- **The `domain=lambda self: [...]` group filter on supervisor/collector** depends on the user actually being in the right `hcpi_data_collection` group. If supervisors disappear from the dropdown, check **Settings → Users** — they probably lost their group assignment.
- **The `mtnce` branches are the active ones.** Kenya and Tanzania each ship two `_18` branches — the plain one is the initial port; `_mtnce` is what's actively maintained. Always check both before assuming a feature isn't there.

## Next

- **[Country Variants](../understanding-the-codebase/country-variants.md)** — the broader branch / overlay / base-module picture across the EAC.
- **[Outlets](outlets.md)** — the generic model every country's `*_outlet` extends.
- **[Data Collection](data-collection.md)** — the base workflow the country overlays modify.
- **[Infrastructure](infrastructure.md)** — `base_import_inherit`, the generic counterpart to the `*_base_import` overlays.
