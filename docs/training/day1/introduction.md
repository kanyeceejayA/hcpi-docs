# Introduction to HCPI

## What HCPI is

HCPI — **Harmonized Consumer Price Index** — is the back-end software statistical bodies in the East African Community (EAC) use to collect, validate, and publish price data for their Consumer Price Index. Uganda's instance ([uboscpi.ubos.org/odoo](https://uboscpi.ubos.org/odoo)) is the reference implementation; other countries adapt it.

It does three big things:

1. **Captures prices in the field.** Enumerators visit outlets (shops, markets) and record prices for predefined items, either through the web UI or the companion **Flutter mobile app**.
2. **Validates and processes the data.** Outliers are flagged by statistical rules (the Tukey algorithm, among others). Validated prices feed into the index computation.
3. **Produces the CPI itself.** Aggregation up the COICOP hierarchy, weighted by basket configuration, with reports and dashboards on top.

## What it's built on

HCPI is not built from scratch — it sits on **[Odoo 18](https://www.odoo.com/documentation/18.0/)**, an open-source business application framework. Odoo gives HCPI the things every line-of-business system needs (ORM, web UI, user management, access control, reporting) so the HCPI team can focus on the price-index logic.

If you've used Odoo before, you'll recognise everything. If you haven't, that's fine — Day 2 onwards walks you through the parts that matter.

```
┌──────────────────────────────────────────────────┐
│              HCPI custom modules                 │  ← What you'll edit
│  hcpi_outlet · hcpi_item · hcpi_data_collection  │
│  hcpi_computation · hcpi_dashboard · ...         │
├──────────────────────────────────────────────────┤
│                    Odoo 18                       │  ← Framework
│      ORM · Web UI · Auth · Access · Reports      │
├──────────────────────────────────────────────────┤
│                  PostgreSQL                      │  ← Data
└──────────────────────────────────────────────────┘
```

## The three things that make an HCPI installation

By the end of today, every trainee's machine has these three things sitting next to each other on disk:

| Piece | What it is | Where it goes |
|---|---|---|
| **Odoo 18** | The framework, cloned from GitHub | `/opt/hcpi/odoo/` |
| **HCPI modules** | The price-index-specific code, from your country's export | `/opt/hcpi/custom/HCPI/` |
| **PostgreSQL database** | Where all the data lives | system-wide, accessed by name (`hcpi`) |

Plus two supporting pieces:

| Piece | Purpose | Where |
|---|---|---|
| **Python virtual environment** | Isolates HCPI's Python dependencies from your system Python | `/opt/hcpi/venv/` |
| **Configuration file** | Tells Odoo where to find addons, the DB, ports, etc. | `/opt/hcpi/conf/hcpi.conf` |

Everything you do today builds toward this layout. The remaining Day 1 pages walk through each piece in turn.

## Architecture — request flow

When a user clicks "Save" on an outlet form, here's roughly what happens:

```
Browser ──HTTP──▶ Odoo (Werkzeug)
                       │
                       ▼
              HCPI custom controller / model
                       │
                       ▼
              Odoo ORM (Python)
                       │
                       ▼
                  PostgreSQL
```

For the mobile app, the entry point is the same Odoo process — the Flutter client speaks a JSON-RPC API rather than rendering HTML. Day 12 covers that side.

## A tour of `custom/HCPI/`

Once installed, your `custom/HCPI/` will contain folders like:

| Module | What it owns |
|---|---|
| `hcpi_coicop` | The COICOP classification hierarchy (used to group items) |
| `hcpi_brand` | Brand reference data |
| `hcpi_item` | Items and units of measure |
| `hcpi_outlet` | Outlets where prices are collected |
| `hcpi_data_collection` | Field data collection forms and observations |
| `hcpi_data_collection_mobile_app` | Mobile-app-facing endpoints |
| `hcpi_computation` | Price relatives, index calculations |
| `hcpi_dashboard` | Dashboards and KPIs |
| `hcpi_index` | The CPI itself — aggregation, periods |
| `ug_location` / `ug_outlet` / `ug_base_import` | Uganda-specific extensions |
| `queue_job` | Background job runner (third-party module HCPI relies on) |

Modules prefixed `ug_` are Uganda's customisations. When your country adopts HCPI you'll usually keep the generic `hcpi_*` modules as-is and create your own `xx_*` modules for country-specific overrides.

The [Understanding the Codebase](../../understanding-the-codebase/index.md) page covers the inside-a-module layout in detail.

## Day 1 goal

By 5pm today, every trainee:

- Has Odoo 18 + HCPI code on their machine
- Can run `python odoo/odoo-bin -c conf/hcpi.conf` and reach `http://localhost:9201` in a browser
- Has VS Code (or PyCharm) connected to the project with the Python interpreter set
- Can log in and knows how to add a new user and reset a password

The remaining Day 1 pages walk through each step, in order.

➡️ Next: [Server Setup](server-setup.md)
