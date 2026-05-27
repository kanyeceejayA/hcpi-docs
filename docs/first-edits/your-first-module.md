# Your First Module

[Making Your First Edits](index.md) walked through changing existing files. This page walks through **creating a whole new module from scratch** — every file Odoo needs, explained as you write it — installing it, using it, and then removing it cleanly.

The module we'll build is intentionally tiny: a "Notes" feature with one model (`hcpi.note`), one list view, one form view, and a menu. About 60 lines of code across six files. By the end you'll have a runnable feature in HCPI and — more importantly — a mental map of what every file in every HCPI module is doing.

!!! info "Before you start"
    - You've read [Odoo Basics](../understanding-the-codebase/odoo-basics.md) — models, fields, views, actions, menus.
    - You've worked through [Making Your First Edits](index.md), or otherwise know the edit → upgrade → refresh loop.
    - HCPI is installed and you can log into `http://localhost:9201`.
    - Odoo is **stopped** for now — we'll start it once the files are in place.

## What we're building

A simple "Notes" feature: a place to jot down dated notes inside HCPI. Each note has a title, body, and date.

When we're done you'll have:

- A new tile in HCPI's app launcher labelled **Notes**
- A list view of your notes, sortable by date
- A form view to create or edit a single note
- The whole thing installed via the normal Odoo "Apps" UI

This isn't useful for actual CPI work — it's a vehicle for showing you the structure of a module. Once you understand it, real modules like `hcpi_outlet` and `hcpi_item` are just bigger versions of the same skeleton.

## The folder skeleton

!!! tip "There's a shortcut — but we won't use it yet"
    Odoo ships a generator: `python odoo/odoo-bin scaffold hcpi_notes /opt/hcpi/custom/HCPI/` creates a starter skeleton in one command. That's how you'll start real modules. But for this first pass we're typing the files by hand so you can see what each one is for. The scaffold shortcut is at the [end of the page](#now-do-it-faster-with-scaffold) — use it on your second module, not your first.

Every Odoo module is a Python package — a folder with `__init__.py` and a `__manifest__.py`. Create this layout under your `custom/HCPI/` directory:

```
custom/HCPI/
└── hcpi_notes/
    ├── __init__.py
    ├── __manifest__.py
    ├── models/
    │   ├── __init__.py
    │   └── hcpi_note.py
    ├── security/
    │   └── ir.model.access.csv
    └── views/
        └── hcpi_note_views.xml
```

```bash
cd /opt/hcpi/custom/HCPI
mkdir -p hcpi_notes/models hcpi_notes/security hcpi_notes/views
cd hcpi_notes
touch __init__.py __manifest__.py
touch models/__init__.py models/hcpi_note.py
touch security/ir.model.access.csv
touch views/hcpi_note_views.xml
```

Now fill each file in turn. The order below is the order Odoo cares about — manifest first (because that's what makes it a module at all), then code, then data.

## 1. `__manifest__.py` — the module's identity card

This single file is what makes a folder a module. Without it, Odoo ignores the folder entirely. Open `hcpi_notes/__manifest__.py` and paste:

```python
{
    'name': 'HCPI Notes',
    'version': '18.0.1.0.0',
    'summary': 'A tiny notes feature — used as a worked example for module structure.',
    'category': 'HCPI',
    'depends': ['base'],
    'data': [
        'security/ir.model.access.csv',
        'views/hcpi_note_views.xml',
    ],
    'installable': True,
    'application': True,
    'license': 'LGPL-3',
}
```

What each key means:

| Key | What it does |
|---|---|
| `name` | The human-readable name shown in the Apps screen. |
| `version` | Convention: `<odoo-major>.<your-major>.<minor>.<patch>` — here `18.0.1.0.0`. |
| `summary` | One-line description in the Apps list. |
| `category` | Where it groups in the Apps screen. Free-form. |
| `depends` | Other modules required. `base` is Odoo's foundation — everything depends on it. If you used `hcpi.outlet`, you'd add `hcpi_outlet` here. |
| `data` | XML/CSV files Odoo should load on install/upgrade. **Order matters** — security first, then views (views can reference security groups). |
| `installable` | Set to `False` to make Odoo ignore the module without deleting it. |
| `application` | `True` makes it appear as a top-level app in the launcher. `False` would tuck it under "Technical". |
| `license` | Required in modern Odoo. `LGPL-3` is the safe default. |

**Why no models in the data list?** Because models are Python — they're loaded automatically through the `__init__.py` chain (next step). Only XML/CSV data files need to be enumerated here.

## 2. The two `__init__.py` files — telling Python what to load

Open `hcpi_notes/__init__.py`:

```python
from . import models
```

Open `hcpi_notes/models/__init__.py`:

```python
from . import hcpi_note
```

These are vanilla Python — they say "when this package is imported, also import these sub-packages/modules". Odoo's module loader imports `hcpi_notes`, which triggers `models/__init__.py`, which triggers `hcpi_note.py`, which defines the model class.

If you forget either `__init__.py` line, your model **silently won't load**. There'll be no error — Odoo just won't know the model exists. This is one of the most common newcomer bugs, so check these two lines if a fresh module installs but its model "doesn't exist."

## 3. The model — `models/hcpi_note.py`

This is the Python class that becomes a database table:

```python
from odoo import fields, models


class HcpiNote(models.Model):
    _name = 'hcpi.note'
    _description = "HCPI Note"
    _order = 'date desc, id desc'

    name = fields.Char(string="Title", required=True)
    content = fields.Text(string="Content")
    date = fields.Date(default=fields.Date.context_today, required=True)
```

Annotating it against what you learned in [Odoo Basics](../understanding-the-codebase/odoo-basics.md):

- `_name = 'hcpi.note'` — the model's technical name. Odoo derives the table name by replacing the dot: `hcpi_note`.
- `_description` — human label. Required in modern Odoo; you get a warning at startup if it's missing.
- `_order = 'date desc, id desc'` — default sort when the ORM reads records. Newest first.
- `name`, `content`, `date` — three fields. `name` is special: Odoo treats it as the record's display label everywhere (breadcrumbs, dropdowns, search bar).
- `default=fields.Date.context_today` — when you click "New", today's date is pre-filled.

That's the whole model. On install, Odoo will create a PostgreSQL table `hcpi_note` with columns `id`, `name`, `content`, `date`, plus the automatic `create_date`, `create_uid`, `write_date`, `write_uid` that every Odoo model gets for free.

## 4. Security — `security/ir.model.access.csv`

**A module with a model but no access rules will refuse to install.** Odoo's security-first stance: if you didn't declare who can touch the table, nobody can, and the install fails with a warning.

Open `security/ir.model.access.csv`:

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_hcpi_note,hcpi.note access,model_hcpi_note,base.group_user,1,1,1,1
```

One row, one rule: any internal user (`base.group_user`) can read/write/create/delete notes.

- `model_id:id` is `model_<table>` — Odoo auto-creates an `ir.model` record for every model with id `model_hcpi_note`.
- The `1`s are booleans for the four CRUD permissions.

In a real module you'd usually have **two** rows — a read-only one for `group_user` and a full-access one for a module-specific manager group. Day 7 of the training covers this properly.

## 5. The views — `views/hcpi_note_views.xml`

This file defines what the user sees. Three pieces in one file: a list view, a form view, an action that opens those views, and two menus that reach the action.

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <!-- List view -->
    <record id="view_hcpi_note_list" model="ir.ui.view">
        <field name="name">hcpi.note.list</field>
        <field name="model">hcpi.note</field>
        <field name="arch" type="xml">
            <list>
                <field name="date"/>
                <field name="name"/>
            </list>
        </field>
    </record>

    <!-- Form view -->
    <record id="view_hcpi_note_form" model="ir.ui.view">
        <field name="name">hcpi.note.form</field>
        <field name="model">hcpi.note</field>
        <field name="arch" type="xml">
            <form>
                <sheet>
                    <group>
                        <field name="name"/>
                        <field name="date"/>
                    </group>
                    <field name="content" placeholder="Write your note..."/>
                </sheet>
            </form>
        </field>
    </record>

    <!-- Action: open the list, with form available for individual records -->
    <record id="action_hcpi_note" model="ir.actions.act_window">
        <field name="name">Notes</field>
        <field name="res_model">hcpi.note</field>
        <field name="view_mode">list,form</field>
        <field name="help" type="html">
            <p class="o_view_nocontent_smiling_face">No notes yet. Click "New" to add the first one.</p>
        </field>
    </record>

    <!-- Top-level menu (the app tile) -->
    <menuitem id="menu_hcpi_note_root"
              name="Notes"
              sequence="100"/>

    <!-- Sub-menu (carries the action) -->
    <menuitem id="menu_hcpi_note"
              name="Notes"
              parent="menu_hcpi_note_root"
              action="action_hcpi_note"
              sequence="1"/>

</odoo>
```

Walk through it top to bottom:

1. **Two views**, each a record in `ir.ui.view`. The `arch` field holds the actual layout XML inside. The list view shows two columns; the form view shows a `<sheet>` (the white-card layout you see everywhere in Odoo) with a header `<group>` and the content as a full-width text area.
2. **One action** (`ir.actions.act_window`) that says "open `hcpi.note` records using the list and form views". Because we didn't specify which views by id, Odoo picks the only ones we've defined.
3. **Two menus**: a root menu (no action — clicking it just opens its children) and a child menu that triggers the action.

Notice every record has an `id` — that's its **XML ID**, the human-readable handle other XML files use to reference it. Fully qualified, it's `hcpi_notes.view_hcpi_note_list` (module name plus local id).

## 6. Install the module

You've written all six files. Now make Odoo see them.

### Start Odoo

```bash
cd /opt/hcpi
source venv/bin/activate
python odoo/odoo-bin -c conf/hcpi.conf
```

### Update the apps list

A new module won't appear in the UI until Odoo scans the addons folders. With **Developer Mode** on (Settings → top of the page → "Activate the developer mode"):

1. Go to **Apps** in HCPI's main menu.
2. Click **Update Apps List** (top bar) → confirm.
3. In the search bar, remove the default "Apps" filter and search for **HCPI Notes**.

Don't see it? See [Troubleshooting](#troubleshooting) below.

### Install

Click the **Activate** button on the **HCPI Notes** tile. Odoo will:

1. Run your Python — create the `hcpi_note` table.
2. Read `ir.model.access.csv` — create the access rule.
3. Read `hcpi_note_views.xml` — write rows into `ir.ui.view`, `ir.actions.act_window`, `ir.ui.menu`.

After about a second, you'll be redirected to the new app. **Notes** is in the main menu. Click **New**, enter a title and some content, save. You've just created a row in `hcpi_note`.

### Verify in the database (optional)

```bash
psql -U hcpi -d hcpi -h localhost -c "SELECT id, name, date FROM hcpi_note;"
```

You'll see the row you just created.

## What just happened, end to end

```
Source files                                      Database (PostgreSQL)
────────────                                      ─────────────────────
hcpi_notes/__manifest__.py     ──install──▶       ir_module_module: installed
hcpi_notes/models/hcpi_note.py ──install──▶       new "hcpi_note" table
                                                  ir_model: model_hcpi_note row
                                                  ir_model_fields: name, content, date rows
security/ir.model.access.csv   ──install──▶       ir_model_access: one row
views/hcpi_note_views.xml      ──install──▶       ir_ui_view: list + form rows
                                                  ir_actions_act_window: one row
                                                  ir_ui_menu: root + child menu rows
```

This is the picture from [Odoo Basics](../understanding-the-codebase/odoo-basics.md#whats-in-the-database-vs-whats-in-the-source) applied to a real install. Every file you wrote produced rows somewhere.

## Iterating after install

The edit → upgrade → refresh loop still applies. If you change `hcpi_note.py` or `hcpi_note_views.xml`, restart Odoo with the upgrade flag:

```bash
python odoo/odoo-bin -c conf/hcpi.conf -u hcpi_notes
```

Or run with `--dev=xml` to live-reload XML view changes without restarting (see [Making Your First Edits](index.md#option-b-run-with-devxml-faster-for-iteration)).

Things worth trying as exercises:

- Add a `priority = fields.Selection([('0', 'Low'), ('1', 'Normal'), ('2', 'High')], default='1')` field, then show it in both views.
- Add a `tags = fields.Char()` field and put it in the search view (`<search><field name="tags"/></search>`).
- Add a Many2one to a user: `author_id = fields.Many2one('res.users', default=lambda self: self.env.user)`.
- Change the menu sequence to move the **Notes** tile around in the app launcher.

Each change is one model edit, one view edit, one `-u hcpi_notes` restart.

## Removing the module

When you're done playing, take it back out cleanly. **Two steps** — uninstall in the UI, then remove the files. Doing only one of them leaves orphan state.

### Step 1: Uninstall via the UI (drops the DB tables)

1. Go to **Apps** in HCPI.
2. Remove the default "Apps" filter, search for **HCPI Notes**.
3. Click the three-dot menu on the tile → **Uninstall** → confirm.

This:

- Deletes the `hcpi_note` table and any data in it.
- Deletes the views, menus, action, and access rules created on install.
- Marks the module as `uninstalled` in `ir_module_module`.

**If you skip this step** and just delete the folder, the module's database records stay behind as orphans. Odoo will warn you on every restart that `hcpi_notes` is referenced but missing on disk.

### Step 2: Remove the folder

Stop Odoo, then:

```bash
rm -rf /opt/hcpi/custom/HCPI/hcpi_notes
```

Start Odoo again. Open **Apps** → **Update Apps List**. The **HCPI Notes** tile is gone.

### Verify it's really gone

```bash
psql -U hcpi -d hcpi -h localhost -c \
  "SELECT name, state FROM ir_module_module WHERE name = 'hcpi_notes';"
```

You should see either no row, or a row with `state = 'uninstalled'`. Either is fine — it means the module isn't active and won't auto-load on next restart.

## Troubleshooting

**Module doesn't appear in Apps after "Update Apps List"**

- Did you put the folder inside `addons_path`? Open `conf/hcpi.conf` and confirm one of the listed paths contains `custom/HCPI/`. If not, edit the conf and restart.
- Is `installable: True` in the manifest? `False` makes Odoo silently skip the module.
- Did you activate Developer Mode? "Update Apps List" is only visible in dev mode.
- Did Odoo print a parse error on startup? Check the terminal — a typo in the manifest or in any XML file will abort the scan.

**Install fails with "missing access rule for model"**

The CSV is wrong. Common slips:

- The model's `_name` is `hcpi.note` but `model_id:id` must reference `model_hcpi_note` (dot → underscore).
- The CSV header line must be exactly the eight columns shown — extra spaces will break the parser.
- File must end with a newline.

**Install succeeds but the menu doesn't appear**

- Hard-refresh your browser (`Ctrl+Shift+R`). Menus are cached client-side.
- Log out and back in.
- Confirm `application: True` in the manifest if you expect a launcher tile (`application: False` puts it under Technical → Menus only).

**`hcpi.note` doesn't exist**

- One of the two `__init__.py` files is missing or empty. Check `hcpi_notes/__init__.py` has `from . import models` and `hcpi_notes/models/__init__.py` has `from . import hcpi_note`.

## Now do it faster with scaffold

Now that you've seen every file by hand, here's the shortcut you'll actually use to start your *next* module:

```bash
cd /opt/hcpi
source venv/bin/activate
python odoo/odoo-bin scaffold hcpi_widgets /opt/hcpi/custom/HCPI/
```

This creates `custom/HCPI/hcpi_widgets/` with a more elaborate skeleton:

```
hcpi_widgets/
├── __init__.py
├── __manifest__.py
├── controllers/         ← HTTP routes (for /portal pages, REST endpoints)
│   ├── __init__.py
│   └── controllers.py
├── demo/                ← seed data loaded only when --without-demo is off
│   └── demo.xml
├── models/
│   ├── __init__.py
│   └── models.py
├── security/
│   └── ir.model.access.csv
├── static/description/  ← icon.png + index.html for the Apps tile
│   ├── icon.png
│   └── index.html
└── views/
    ├── templates.xml    ← website/QWeb templates
    └── views.xml
```

You now know what every one of those folders is for — controllers, demo, static/description, views — because you wrote the minimal version yourself. Open the generated files and you'll recognise everything.

**What to do with the extra folders:** for a model-only module like `hcpi_notes`, you'd delete `controllers/` and `views/templates.xml` (you don't have HTTP routes or website templates). For a real HCPI module, you'll typically keep `models/`, `security/`, `views/`, `static/description/`, and drop the rest until you need them.

The takeaway from this whole page: **scaffold gives you the skeleton; understanding lets you trim it.**

## What you learned

In one tutorial:

- The seven files that make a minimal Odoo module work, and what each one contributes.
- Why a missing `__init__.py` line is silent — and how to spot it.
- Why a module without security rules refuses to install.
- The full path from a folder in `custom/HCPI/` to rows in PostgreSQL tables, and back out again on uninstall.

Real HCPI modules (`hcpi_outlet`, `hcpi_item`, `hcpi_index`) are this same skeleton with **more** models, **more** views, controllers, reports, computed fields, related modules — but the bones are identical. When you open one of them next, you'll recognise every file.

## What's next

- **[Building an HCPI Module — Part 1](../training/module/part1-models.md)** — the proper guided tutorial. Builds a realistic, multi-model module (outlet onboarding) and walks through every concept you'll meet in HCPI: relationships, computed fields, all view types, security, workflow, chatter.
- **[Module Reference](../understanding-the-codebase/module-reference.md)** — what each real HCPI module actually does.
- **[Country Variants](../understanding-the-codebase/country-variants.md)** — how country-specific modules extend the base HCPI modules using `_inherit`.
