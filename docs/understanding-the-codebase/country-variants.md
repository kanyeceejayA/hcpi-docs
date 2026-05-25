# Country Variants

The HCPI source repo holds **one branch per country deployment**. The `c:/hcpi` checkout you've been working with is one country's worldview (Uganda); the same shared core lives in every other country's branch with a different overlay on top.

This page maps the branch zoo, the two architectural eras (modern overlay vs. legacy single-folder), and the concrete differences between country overlays — what each country adds, where the location hierarchies differ, and why some countries need a `xx_data_collection` overlay while others don't.

Source: [github.com/kola-tech/HCPI-full](https://github.com/kola-tech/HCPI-full) (the canonical multi-branch repo). All findings below come from `git ls-tree`/`git show` against that repo.

## The branch zoo

Sixteen branches in current use. Read the suffixes:

| Suffix pattern | Meaning |
|---|---|
| `18.0` | The shared upstream — Odoo 18 base, no country overlay |
| `xx_18` | Country `xx` deployment, currently on Odoo 18 |
| `xx_18_mtnce` | Maintenance branch for `xx_18` (release candidates / hotfixes that merge back into `xx_18`) |
| `xx` (no number) | **Legacy** — pre-Odoo 18 deployment, single-folder architecture, frozen since March 2024 |
| `xx-18` | Old intermediate naming — superseded by `xx_18` |
| `ft-...` | Feature branches (e.g. `ft-imputations_ug`, `ft-ug-outliers`) |
| `secretariat` | EAC Secretariat regional aggregator (a completely different application) |
| `main` | Historical default — superseded by `18.0` as the upstream |

### Active country branches

| Country | Active branch | Last commit | Status |
|---|---|---|---|
| **Upstream** | `18.0` | 2026-04-14 | Shared base |
| **Uganda** | `ug_18` | 2026-04-21 | Active |
| **Kenya** | `ke_18_mtnce` | 2025-07-07 | Maintenance — last release rolled up here |
| **Tanzania** | `tz_18_mtnce` | 2025-07-09 | Maintenance |
| **Zanzibar** | `zar_18` | 2026-04-14 | Active |
| **EAC Secretariat** | `secretariat` | 2026-04-20 | Active (separate aggregator app) |

### Legacy branches (frozen 2024)

| Country | Branch | Last commit | Notes |
|---|---|---|---|
| Rwanda | `rw` | 2024-03-13 | Old single-folder architecture |
| South Sudan | `ss` | 2024-03-13 | Old single-folder architecture |
| Burundi | `bi` | 2024-03-13 | Old single-folder architecture |

All three legacy branches received the same trivial last commit on the same day ("hiding hcpi menu in settings"). They predate the consolidation onto a shared core.

!!! warning "`zar` = Zanzibar, not South Africa"
    Despite "ZAR" being the ISO currency code for South African Rand, in this repo `zar`/`zar_18` is **Zanzibar** (semi-autonomous part of Tanzania). The `zar_data_collection` manifest is explicit: *"Customize and manage Zanzibar data collections"*. South Africa isn't part of the HCPI deployment family.

## Two architectures

There's a clean break between branches written before and after the codebase was consolidated.

### Modern overlay (UG, KE, TZ, ZAR, plus the shared `18.0`)

```
18.0 (upstream)                    Country branch (e.g. ug_18)
─────────────────                  ────────────────────────────
hcpi_coicop                        hcpi_coicop          ◀── inherited verbatim
hcpi_item                          hcpi_item            ◀── inherited verbatim
hcpi_outlet                        hcpi_outlet          ◀── inherited verbatim
hcpi_computation                   hcpi_computation     ◀── inherited verbatim
hcpi_data_collection               hcpi_data_collection ◀── inherited (mostly verbatim)
hcpi_data_collection_mobile_app    hcpi_data_collection_mobile_app
hcpi_index                         hcpi_index           ◀── inherited verbatim
hcpi_dashboard                     hcpi_dashboard       ◀── inherited verbatim
hcpi_brand                         hcpi_brand           ◀── inherited verbatim
base_import_inherit                base_import_inherit  ◀── inherited verbatim
                                   ┌─────────────────────────┐
                                   │ ug_location  ◀── ADDED  │
                                   │ ug_outlet    ◀── ADDED  │
                                   │ ug_base_import ◀── ADDED│
                                   └─────────────────────────┘
```

Country overlays only *add* `xx_*` modules. The shared core modules are pulled in verbatim from `18.0` via regular git merges (visible as `Merge pull request #129 from kola-tech/18.0` in the branch history). When upstream changes — a bug fix in `hcpi_outlet`, say — every country branch can merge it without losing their overlay.

This is the pattern the [Module Reference](module-reference.md) describes. It's how new countries should be added.

### Legacy single-folder (RW, SS, BI — frozen)

```
hcpi-rw/
├── rw_hcpi_coicop/        ← COICOP, but country-specific
├── rw_locations/          ← geography
├── rw_outlets/            ← outlets
├── rw_products/           ← items (called "products" here)
├── rw_data_collections/   ← collection
├── rw_dashboard/          ← per-country dashboard
├── rw_matrix/             ← Rwanda-only: pivot view of price relatives
├── rw_price_sheets/       ← Rwanda-only: price sheet workflow
├── rw_data_collection_mobile_app/
└── base_import_inherit/
```

Everything is country-prefixed — there's no shared core to merge from. RW, SS, and BI all have nearly the same folder structure (Burundi and Rwanda also have `xx_matrix`; South Sudan doesn't). These predate the decision to split country-specific logic from the shared HCPI domain model.

**If you're adopting HCPI for a new country, do not use this pattern.** Use the modern overlay (see [Module Reference](module-reference.md)).

## Country overlays compared

Every active country (modern branches) ships the same template: a `xx_location` module defining the geographic hierarchy, a `xx_outlet` module linking outlets to that geography, and optionally `xx_data_collection` and `xx_base_import` modules.

But the **hierarchies are different**, and **not every country needs every overlay**.

### Geographic hierarchies

| Country | Levels | Models |
|---|---|---|
| **Uganda** | 5 | `ug.region` → `ug.district` → `ug.county` → `ug.sub.county` → `ug.parish` |
| **Kenya** | 3 | `ke.region` → `ke.county` → `ke.zone` |
| **Tanzania** | 2 | `tz.region` → `tz.district` |
| **Zanzibar** | 3 | `zar.region` → `zar.district` → `zar.collection.center` |

Hierarchy depth follows each country's administrative structure. Uganda is the deepest (parish-level data collection); Tanzania is the flattest (district-level).

### Outlet anchor

Each `xx_outlet` module extends `hcpi.outlet` with the **deepest** geographic FK; the higher levels are stored-related fields that cascade up the chain for fast filtering.

| Country | `_inherit` anchor field |
|---|---|
| Uganda | `parish_id` (the leaf); `sub_county_id`/`county_id`/`district_id`/`region_id` are related-stored |
| Kenya | `zone_id` (the leaf); `county_id`/`region_id` are related-stored |
| Tanzania | `district_id` (the leaf); `region_id` is related-stored |
| Zanzibar | `collection_center_id` (the leaf); `district_id`/`region_id` are related-stored |

So filtering "all outlets in Kampala" works the same way in every country — just substitute the right field name.

### Which extra modules each country needs

Not every country has every overlay. Some countries needed extra workflow customization (bulk questionnaire wizards, specific validation flows); others use the core HCPI behaviour as-is.

| Module pattern | Uganda | Kenya | Tanzania | Zanzibar |
|---|---|---|---|---|
| `xx_location` | ✅ | ✅ | ✅ | ✅ |
| `xx_outlet` | ✅ | ✅ | ✅ | ✅ |
| `xx_data_collection` | ❌ | ✅ | ✅ | ✅ |
| `xx_base_import` | ✅ | ✅ | ✅ | ❌ |

**Why Uganda has no `ug_data_collection`:** the core `hcpi_data_collection` module was largely written against Uganda's workflow, so Uganda's needs are covered by the base behaviour. Other countries added their own variants where their workflow diverges.

**What `xx_data_collection` typically adds:**

| Country | Models added |
|---|---|
| `ke_data_collection` | `bulk_questionnaire_wizard`, extends `hcpi.data.collection`, extends `hcpi.data.validation` |
| `tz_data_collection` | `bulk_questionnaire_wizard`, `assign_questionnaire_wizard`, extends `hcpi.data.collection`, extends `hcpi.data.validation` |
| `zar_data_collection` | bulk + assign questionnaire wizards (in `wizards/`), extends `hcpi.data.collection` |

Common themes: **bulk-creation wizards** for spinning up many questionnaires at once, and **assignment wizards** for batching surveys to enumerators.

## Has the core drifted across branches?

In a perfect world, the core HCPI modules (`hcpi_outlet`, `hcpi_data_collection`, `hcpi_index`, etc.) would be **identical** between `18.0` and every country branch — country deltas would live entirely in the `xx_*` overlay modules.

In practice, some drift has accumulated:

| Branch | Diff in `hcpi_data_collection` vs. `18.0` |
|---|---|
| `ug_18` | None — fully in sync |
| `zar_18` | None — fully in sync |
| `ke_18_mtnce` | 28 files changed, mostly file moves and renames |
| `tz_18_mtnce` | 9 files changed, includes small Python tweaks |

Kenya and Tanzania are on `_mtnce` branches that haven't merged the latest `18.0` yet — once they do, the drift should close. If you're inheriting maintenance of one of these branches, expect to do a `git merge 18.0` and resolve a handful of conflicts before adding new features.

## The EAC Secretariat — separate application

The `secretariat` branch is **not** a country deployment. It's a regional aggregator: one Odoo instance that pulls CPI snapshots from each country's HCPI database over XML-RPC and presents a consolidated dashboard.

Folder layout:

```
secretariat/
└── hcpi_secretariat_hub/
    ├── __manifest__.py
    ├── models/
    │   ├── remote_instance.py
    │   ├── snapshot.py
    │   ├── dashboard.py
    │   └── res_users.py
    ├── data/
    │   └── ir_cron.xml          ← scheduled syncs
    ├── views/
    ├── security/
    └── static/                  ← OWL dashboard
```

Only one HCPI-aware module: `hcpi_secretariat_hub`. It depends only on Odoo's `base` and `web` — none of the country modules — because it doesn't share data tables with them; it talks to them as remote clients.

### Key models

| Model | Purpose |
|---|---|
| `hcpi.secretariat.instance` | One per country deployment. Fields: `name`, `code` (KE/TZ/UG/ZAR), `base_url`, `database_name`, `username`, `api_key`, `sync_month_window` (default 12), `state` (draft/connected/error), `last_sync_at`, `last_sync_status`. Unique on `code`. |
| `hcpi.secretariat.sync.log` | One row per sync attempt. Fields: `instance_id`, `status` (running/success/failed), `started_at`, `finished_at`, `records_synced`, `message`. |
| `hcpi.secretariat.country.snapshot` | The actual aggregated data. One row per (country, month). Fields: `instance_id`, `country_code` (stored-related), `snapshot_month`, `current_cpi`, `min_cpi`, `max_cpi`, `total_outlets`, `total_items`, `total_outlet_items`, `total_observations`, `last_sync_at`. |

### How it works

1. An admin creates `hcpi.secretariat.instance` records — one per country, with that country's Odoo URL and an API key for an integration user.
2. An `ir.cron` job runs periodically (defined in `data/ir_cron.xml`), iterates instances, opens an XML-RPC connection to each, and pulls the latest N months of indicators.
3. Each pull writes a `sync.log` and (on success) one or more `country.snapshot` rows.
4. The OWL dashboard (`static/src/js/secretariat_dashboard.js`) reads from `country.snapshot` to render the regional view.

### Why XML-RPC and not direct DB access?

Two reasons. Each country runs its HCPI in its own data centre under its own authority — the secretariat has no business connecting to their PostgreSQL directly. And the indicators are derived from queries that already exist in `hcpi_index`; calling them via RPC reuses that logic, rather than re-implementing index computation in the aggregator.

The integration user on each country instance is granted read access to the relevant index models only. If you're standing up a new country and integrating it with the secretariat, you'll need to create that user and share the API key.

## Adopting HCPI for a new country

The recipe, distilled from how UG/KE/TZ/ZAR are structured:

1. **Create a branch off `18.0`** named `xx_18` (e.g. `gh_18` for Ghana).
2. **Create `xx_location/`** — define your country's administrative hierarchy as a chain of models. Match the existing pattern: one model per level, `code` and `name` fields, `mail.activity.mixin` for tracking.
3. **Create `xx_outlet/`** — `_inherit = 'hcpi.outlet'`, add a Many2one to the leaf level of your geography, and stored-related fields up the chain.
4. **(Optional) Create `xx_data_collection/`** — only if your country's workflow diverges from the core. Typical adds: bulk-creation wizards, country-specific validation rules.
5. **(Optional) Create `xx_base_import/`** — only if your country has unusual import formats or validation rules.
6. **Update `hcpi.conf`** on the deployment server: add the new module folders to `addons_path`.
7. **For secretariat integration:** create an integration user on the new instance with read access to indices; share base URL + database name + API key with the secretariat team.

Avoid:

- ❌ Copying the **legacy single-folder** layout (`hcpi-xx/`). It's frozen for a reason.
- ❌ Modifying core `hcpi_*` modules in your country branch — the drift will haunt you at merge time.
- ❌ Naming a new branch `xx` (no number) — that pattern is associated with legacy branches and is confusing.

## Quick reference

| If you want to... | Look in... |
|---|---|
| See what's specific about Uganda's outlets | `ug_18:ug_outlet/models/hcpi_outlet.py` |
| Compare Kenya's vs Tanzania's geography depth | `ke_location/models/` vs `tz_location/models/` |
| Find Zanzibar's bulk-questionnaire wizard | `zar_18:zar_data_collection/wizards/` |
| Inspect Rwanda's old architecture | `rw:hcpi-rw/` (read-only — frozen) |
| See what the secretariat aggregates | `secretariat:hcpi_secretariat_hub/models/snapshot.py` |
| See how country branches merge from upstream | `git log --merges 18.0..ug_18` (etc.) |
