# Odoo Basics

Before you start editing HCPI, you need a working mental model of how Odoo is put together. This page explains the handful of concepts everything else in the docs assumes: **models, fields, records, views, actions, menus**, and how they live in the database vs. in the source files.

If you've used Odoo before, skim it. If not, read it carefully — it'll save you confusion later.

## The four-second tour

A user clicks "Outlets" in HCPI's app launcher. Here's what happens in your head:

```
  Menu  ──▶  Action  ──▶  View  ──▶  Model  ──▶  Database
("Outlets")  (open list)  (list/    (hcpi.outlet)  (PostgreSQL)
                          form)
```

That's the whole chain. Every clickable thing in Odoo's UI follows this path. The rest of the page expands each box.

## Models — the database side

A **model** in Odoo is a Python class that becomes a database table. Each instance of the class is a **record** (a row). Each attribute of the class is a **field** (a column).

From `hcpi_outlet/models/hcpi_outlet.py`:

```python
class Outlet(models.Model):
    _name = 'hcpi.outlet'
    _description = "Outlet"
    _inherit = ['mail.thread', 'image.mixin']

    name = fields.Char(required=True)
    outlet_code = fields.Char(required=True)
    consumption_segment_id = fields.Many2one('hcpi.consumption.segment')
    latitude = fields.Float()
    outlet_item_ids = fields.One2many('hcpi.outlet.item', 'outlet_id')
```

This declaration produces a PostgreSQL table — its name is the model's `_name` with dots replaced by underscores: `hcpi_outlet`. Connect to the DB and you can see it:

```bash
psql -U hcpi -d hcpi -h localhost -c "SELECT id, name, outlet_code FROM hcpi_outlet LIMIT 5;"
```

| Concept | What it is | Example |
|---|---|---|
| **Model** | A Python class that maps to a DB table | `hcpi.outlet` |
| **Record** | One row in that table | A specific outlet, e.g. "Kikuubo Wholesale Market" |
| **Field** | A column on the table | `name`, `outlet_code`, `latitude` |
| **Recordset** | Zero or more records of the same model, treated as one object | `self.env['hcpi.outlet'].search([])` returns all outlets |

### Field types you'll meet

| Field type | What it stores | PostgreSQL column |
|---|---|---|
| `Char(...)` | Short string | `VARCHAR` |
| `Text(...)` | Long string | `TEXT` |
| `Integer(...)` | Whole number | `INTEGER` |
| `Float(...)` | Decimal | `NUMERIC` |
| `Boolean(...)` | True/false | `BOOLEAN` |
| `Date(...)` / `Datetime(...)` | Date / timestamp | `DATE` / `TIMESTAMP` |
| `Selection([(...)])` | Pick from a fixed list | `VARCHAR` with check |
| `Many2one('other.model')` | FK to one record of another model | `INTEGER` (the other table's id) |
| `One2many('other.model', 'inverse_field')` | Reverse side of a Many2one — virtual, no DB column | (none) |
| `Many2many('other.model')` | Many-to-many via a join table | Join table |
| `Binary(...)` | Files, images | `BYTEA` |
| `Selection([('a', 'A')])` | Enum-like | `VARCHAR` |
| `Compute=...` (any of the above) | Computed in Python from other fields — can be stored or not | depends on `store=True` |

One2many fields **don't have a column** — they're computed by looking at the inverse Many2one on the other model. That's why declaring a One2many requires you to name the field on the other side that points back.

### `_inherit` — extending an existing model

A second class can extend an existing model:

```python
# In ug_outlet/models/hcpi_outlet.py
class Outlet(models.Model):
    _inherit = 'hcpi.outlet'

    parish_id = fields.Many2one('ug.parish')
    district_id = fields.Many2one('ug.district', related='parish_id.district_id', store=True)
```

This adds two new columns to the existing `hcpi_outlet` table. It does **not** create a new table — `ug_outlet` overlays `hcpi.outlet` rather than copying it. This is the country-customization pattern from the [Module Reference](module-reference.md).

## Views — what the user sees

A **view** is an XML definition of a piece of UI. Views are stored as records in a built-in Odoo table called `ir.ui.view`, and they reference the model they're for.

Open `hcpi_outlet/views/hcpi_outlet_views.xml` and you'll see three views for outlets: a **list view** (used to show many outlets at once), a **form view** (used to show one outlet for editing), and a **search view** (used for the filter/group sidebar).

### The five common view types

| View type | XML tag | What it shows |
|---|---|---|
| **List** | `<list>` | A table — many records at once, one row each |
| **Form** | `<form>` | A single record's fields laid out for viewing/editing |
| **Search** | `<search>` | The search bar with filters and "Group By" options |
| **Kanban** | `<kanban>` | Cards (think Trello), grouped by some field |
| **Graph** | `<graph>` | Charts (bar, pie, line) over aggregated data |

There are a few more — Pivot, Calendar, Gantt — but the five above cover 95% of what HCPI uses.

### A view is *just a record*

This is the part newcomers miss: views are themselves data. They live in `ir.ui.view`. The XML file isn't *executed* by Odoo at runtime — when you install or update a module, Odoo reads the XML and writes records into `ir.ui.view`. From then on, Odoo renders views by **reading them out of the database**, not by reading the XML again.

That's why changing a view file requires either:

- `-u <module>` to re-run the install/upgrade and overwrite the DB records, or
- `--dev=xml` mode, which makes Odoo reload XML from disk on each request (great for dev iteration).

The same is true for actions, menus, security rules, email templates — they're all records in their respective tables.

### "Tree" means "list"

Odoo historically called list views `<tree>`. Odoo 17/18 prefer `<list>`, but you'll see both in older code and in stack traces. If a Python error mentions a "tree view", it means **list view** — not a hierarchical tree.

## Actions — what happens when you click

An **action** describes what to do when a user clicks something. There are several kinds; the most common is `ir.actions.act_window`, which says "open a window showing records of model X using views Y".

From `hcpi_outlet/views/hcpi_outlet_views.xml`:

```xml
<record id="hcpi_outlet_action" model="ir.actions.act_window">
    <field name="name">Outlets</field>
    <field name="res_model">hcpi.outlet</field>
    <field name="view_mode">list,form</field>
    <field name="domain">[]</field>
    <field name="help" type="html">
        <p class="o_view_nocontent_smiling_face">Create an Outlet!</p>
    </field>
</record>
```

This action:

- Targets the `hcpi.outlet` model
- Opens **list view first**, with form view available when you click a record
- Applies no filter (`domain` is empty)
- Shows a friendly message when the list is empty

Actions are records too — they're stored in `ir.actions.act_window` (with `model="ir.actions.act_window"` in the XML).

### Common action types

| Action model | What it does |
|---|---|
| `ir.actions.act_window` | Opens a list/form view for a model |
| `ir.actions.server` | Runs Python code on the server (no UI) |
| `ir.actions.client` | Opens a JavaScript-defined client view (dashboards, settings) |
| `ir.actions.report` | Generates a PDF/HTML report |
| `ir.actions.act_url` | Redirects the browser to a URL |

## Menus — the navigation

A **menu** is a clickable label in the navigation that triggers an action.

From `hcpi_outlet/views/hcpi_outlet_menus.xml`:

```xml
<menuitem id="menu_hcpi_outlet_root"
          name="Outlets"
          web_icon="hcpi_outlet,static/description/icon.png"
          sequence="25"
          groups="hcpi_outlet.group_outlet_manager" />

<menuitem id="menu_hcpi_outlet"
          name="Outlets"
          sequence="1"
          parent="menu_hcpi_outlet_root"
          action="hcpi_outlet_action" />
```

Reading these:

1. The **root menu** — top-level, appears in HCPI's app launcher with an icon. It doesn't have an action; clicking it just opens its children.
2. The first **sub-menu** — sits inside the root, has `action="hcpi_outlet_action"` (the action above). Clicking this is what opens the outlets list.

`groups="..."` restricts who can see the menu. Without that group, the menu doesn't appear in the user's sidebar.

Menus are records in the `ir.ui.menu` table.

## Security — who can do what

Two layers, both ultimately records in the database:

### `ir.model.access.csv` — model-level CRUD

A CSV file in `security/` listing which group can perform create/read/update/delete on which model:

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_hcpi_outlet_manager,hcpi.outlet.manager,model_hcpi_outlet,group_outlet_manager,1,1,1,1
access_hcpi_outlet_user,hcpi.outlet.user,model_hcpi_outlet,group_outlet_user,1,0,0,0
```

Two rows — one says managers can do everything to outlets, another says regular users can only read them.

### `ir.rule` — record-level filtering

If you need "users can only see outlets in their own region" — that's a **record rule** (`ir.rule`), defined in XML. It's a domain expression Odoo automatically adds to every query against that model.

Day 7 of the training covers security in depth.

## What's in the database vs. what's in the source

The two-side picture:

```
SOURCE FILES                          DATABASE (PostgreSQL)
────────────────                      ─────────────────────
Python model classes        ──install/upgrade──▶   hcpi_outlet table, hcpi_item table, ...
XML <record> for views      ──install/upgrade──▶   ir_ui_view rows
XML <record> for actions    ──install/upgrade──▶   ir_actions_act_window rows
XML <menuitem>              ──install/upgrade──▶   ir_ui_menu rows
security/ir.model.access.csv ──install/upgrade──▶  ir_model_access rows
XML <record> for security rules ──install/upgrade──▶ ir_rule rows
data XML (seed records)     ──install/upgrade──▶   their respective tables

USER DATA (live, not in source):
                                                   outlets created in the UI
                                                   observations recorded by enumerators
                                                   computed indices, reports, etc.
```

The takeaway: **the source files describe the structure; the database holds the structure plus the data**. When you change a source file, nothing happens until you run `-u <module>` to push the changes into the DB.

### Tables you'll see when you poke at the DB

Most of these are Odoo's metadata tables — the "system tables" that store the structure itself:

| Table | What's in it |
|---|---|
| `ir_module_module` | All modules and their install state |
| `ir_model` | Every model in the system (`hcpi.outlet`, `res.users`, ...) |
| `ir_model_fields` | Every field on every model |
| `ir_ui_view` | Every view |
| `ir_ui_menu` | Every menu |
| `ir_actions_act_window` | Every "open this list" action |
| `ir_model_access` | The CRUD permissions |
| `ir_rule` | Record-level filtering rules |
| `res_users` | User accounts |
| `res_groups` | Permission groups (`group_outlet_manager`, etc.) |

Then HCPI's own tables: `hcpi_outlet`, `hcpi_item`, `hcpi_outlet_item`, `hcpi_outlet_item_observation`, `hcpi_national_index`, and so on.

```bash
# Quick look at what's installed
psql -U hcpi -d hcpi -h localhost -c "SELECT name, state FROM ir_module_module WHERE name LIKE 'hcpi%' OR name LIKE 'ug_%';"

# How many views are defined for outlets?
psql -U hcpi -d hcpi -h localhost -c "SELECT name, type FROM ir_ui_view WHERE model = 'hcpi.outlet';"
```

## The ORM — your interface to the database

You almost never write raw SQL in Odoo. Instead, models expose methods that translate to SQL behind the scenes. The core five:

| Method | What it does | SQL equivalent |
|---|---|---|
| `search(domain)` | Find records matching a filter | `SELECT id FROM ... WHERE ...` |
| `browse(ids)` | Fetch known records by id | `SELECT * FROM ... WHERE id IN (...)` |
| `read([fields])` | Read fields from records | `SELECT field, ... FROM ...` |
| `create(values)` | Insert a new record | `INSERT INTO ...` |
| `write(values)` | Update existing records | `UPDATE ... SET ...` |
| `unlink()` | Delete records | `DELETE FROM ...` |

`search` returns a **recordset** — a list-like object whose elements are records. Recordsets are chainable:

```python
# All active outlets in Kampala
outlets = self.env['hcpi.outlet'].search([
    ('active', '=', True),
    ('district_id.name', '=', 'Kampala'),
])

# Loop and modify
for outlet in outlets:
    outlet.write({'last_visited': fields.Date.today()})

# Or batch — one UPDATE for all
outlets.write({'last_visited': fields.Date.today()})
```

The `domain` (the `[('field', '=', 'value'), ...]` syntax) is Odoo's polish on SQL `WHERE` clauses. Day 3 of the training covers domains and ORM methods properly.

## The Python ↔ XML loop

Most HCPI features are built by writing **both**:

1. A model in Python (`models/whatever.py`) — defines the table and the business logic
2. Views in XML (`views/whatever_views.xml`) — defines how the model shows up in the UI
3. An action and a menu in XML — to make it reachable

That's why almost every module has parallel `models/` and `views/` folders. When you add a new field, you typically edit a file in both.

## Developer mode — your superpower

Once you've turned on developer mode (covered on the [User Administration](../training/day1/user-administration.md) page), the UI exposes the underlying records:

- **Hover any field** with developer mode on → tooltip shows the model name and field name (the technical handles you'll search for in code).
- **The Developer menu** (bug icon, top-right) → "Edit View", "Edit Action", "Manage Models". Each opens the underlying `ir.ui.view`/`ir.actions.act_window`/`ir.model` record directly in a form.
- **URL parameters** become readable: `/odoo/action-hcpi_outlet.hcpi_outlet_action` tells you exactly which action you're looking at.

This is how you go from "I see something on screen and want to change it" to "I know the model name, the view ID, and the file it's defined in." Combined with project-wide search in your IDE, that's the loop you'll spend most of your time in.

## Advanced: QWeb and OWL — the two frameworks you'll see

Two pieces of Odoo's stack are worth naming so you recognise them when they appear. You don't need to *write* either to do most HCPI work — but you'll *read* them, and you'll occasionally need to know which one you're looking at.

### QWeb — Odoo's templating language

QWeb is **HTML with control-flow attributes** prefixed `t-*`. It looks like this:

```xml
<table>
    <tr t-foreach="docs" t-as="proposal">
        <td><span t-field="proposal.outlet_name"/></td>
        <td t-if="proposal.has_failed_visit" class="text-danger">⚠</td>
    </tr>
</table>
```

The common attributes you'll meet:

| Attribute | What it does |
|---|---|
| `t-foreach` + `t-as` | Loop over a collection, binding each item to a name. |
| `t-if` / `t-elif` / `t-else` | Conditionals. |
| `t-field="record.fieldname"` | Render a field's value with its built-in formatting (dates, monetary, etc.). |
| `t-out="expression"` | Render a Python expression. |
| `t-call="template.id"` | Include another template (like a partial). |
| `t-att-class="..."` / `t-attf-class="..."` | Compute an attribute. `t-att-` takes an expression; `t-attf-` takes a string with `#{...}` substitutions. |

**Two runtimes, same syntax.** QWeb is evaluated in two places:

- **Server-side** for **PDF reports** (`ir.actions.report` + `report_type="qweb-pdf"`), **website pages**, and any place Python pre-renders HTML.
- **Client-side** by **OWL** (below) for **kanban cards**, **dashboards**, and dynamic view fragments.

You write the same template; Odoo picks the runtime based on context.

**Where HCPI uses it:**

- **Kanban card layouts** — every `<templates><t t-name="card">` block in a `<kanban>` view is QWeb.
- **PDF outputs** — country-specific report templates (basket reports, index print-outs).
- **Email templates** in `hcpi_data_collection` and `kola_web_enterprise`.

When you see a `.xml` file under `views/` or `reports/` with lots of `t-foreach` and `t-field`, that's QWeb. Reference: <https://www.odoo.com/documentation/18.0/developer/reference/frontend/qweb.html>.

### OWL — Odoo's frontend framework

**OWL** (Odoo Web Library) is Odoo's reactive component framework. Conceptually similar to Vue or React but tuned for Odoo's needs. It's what renders every form, list, kanban, statusbar, and dialog in the web client.

A component looks like this:

```javascript
/** @odoo-module */
import { Component } from "@odoo/owl";
import { registry } from "@web/core/registry";

class MyBadge extends Component {
    static template = "my_module.MyBadge";
    get cssClass() {
        return this.props.record.data.state === "active"
            ? "text-bg-success"
            : "text-bg-secondary";
    }
}

registry.category("fields").add("my_badge", { component: MyBadge });
```

The template (`my_module.MyBadge`) is — yes — a QWeb template, registered in an XML file under the module.

**Where it lives:** OWL code goes in `static/src/` inside a module. JavaScript files end in `.js`; their companion templates are `.xml`. Both are bundled into Odoo's asset pipeline via the manifest's `assets` key.

**When you'd write OWL yourself:**

- A **custom field widget** — e.g. an interactive map for the `latitude`/`longitude` fields on an outlet.
- A **custom view type** or override of an existing renderer.
- A **client action** — a fully bespoke UI screen, often a dashboard, declared as `ir.actions.client`.

**Where HCPI uses it:** mostly invisibly — every UI element you've used so far is an OWL component shipped by Odoo core. HCPI's own JS surface area is small: a few menu / branding tweaks under `kola_web_enterprise/static/src/`, the dashboard widgets in `hcpi_dashboard`, and the occasional small field widget. The other 95% of HCPI's UI comes from declaring `<list>`, `<form>`, `<kanban>` records and letting Odoo's built-in OWL components do the rendering.

When you find yourself looking at a `.js` file under `static/src/` with `import { Component }`, that's OWL. Reference: <https://github.com/odoo/owl#readme>.

### How to tell them apart at a glance

| File location | Likely framework |
|---|---|
| `views/*.xml` with `<list>`, `<form>`, `<kanban>` records | View definitions — rendered by Odoo's built-in OWL components |
| `views/*.xml` with `<templates>` + `t-foreach` | **QWeb** (kanban card layout, mostly) |
| `reports/*.xml` with `<template>` + `t-call="web.external_layout"` | **QWeb** (PDF report) |
| `static/src/**/*.js` | **OWL** (component code) |
| `static/src/**/*.xml` paired with a `.js` of the same name | **QWeb** (template for an OWL component) |

## What you should remember

- **Models** are Python classes → database tables; **records** are rows; **fields** are columns.
- **Views, actions, menus, security rules** are all stored as records in built-in `ir_*` tables. The XML in your source files writes those records on install/upgrade.
- **"Tree" means "list"** in old XML.
- Editing a view file does nothing until you `-u <module>` (or have `--dev=xml` on).
- The **ORM** is how you read and write data without raw SQL.
- **Developer mode** is how you go from UI to code.

## Next

➡️ **[Understanding the Codebase](index.md)** — the folder map: which folder, which file, when.

➡️ **[Module Reference](module-reference.md)** — what each HCPI module does and which models it owns.

➡️ **[Making Your First Edits](../first-edits/index.md)** — apply this with a real change you can see.

➡️ **[Your First Module](../first-edits/your-first-module.md)** — build a tiny end-to-end module from scratch and see every file from this page in context.
