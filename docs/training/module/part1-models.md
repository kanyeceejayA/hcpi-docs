# Building HCPI Field Reports — Part 1: Models, Fields, Relationships

**Duration:** half a day (about 4 hours including breaks).
**Covers:** models, fields and field types, relationships (Many2one, One2many, Many2many, hierarchical), computed fields, model inheritance, the ORM, minimal views to display the data.

This is the start of a three-part hands-on tutorial. You'll build **`hcpi_field_reports`** — a small but realistic HCPI module where enumerators submit daily reports after going out to collect prices, and supervisors review them. By the end of all three parts you'll have touched every basic concept in the Day 3–7 programme.

!!! info "Before you start"
    - You've worked through [Your First Module](../../first-edits/your-first-module.md). You don't need to remember every detail — you just need to know what a manifest, model, view, and security file are.
    - You've read [Python for the HCPI Platform](../day2/python-intro.md) (or you're comfortable enough with Python that comprehensions, classes, and decorators don't slow you down).
    - HCPI is installed and running on `http://localhost:9201`.

## What we're building (across all three parts)

A field-activity reporting module. The story:

1. An enumerator finishes a day in the field and opens HCPI.
2. They create a **Field Report** with the date, region, hours worked, outlets visited, prices collected.
3. They list any **observations** — broken refrigerators, missing items, market closures.
4. They tag the report (`weekend`, `rainy-day`, `mobile-app`, etc.) and submit it.
5. A **supervisor** reviews submitted reports, approves or rejects, comments on issues.
6. Managers see **graphs and pivots** across regions and time periods.

What each part builds:

| Part | Focus | What you end up with |
|---|---|---|
| **1 (this page)** | Models, fields, relationships, ORM | A working module with three models, list and form views, data you can create and query. |
| **[2: Views & UX](part2-views.md)** | All view types, search, reports | Kanban with stages, graphs, pivot, calendar, filters, a printable PDF. |
| **[3: Security & Polish](part3-security.md)** | Groups, access rules, workflow, chatter | Production-ready: roles, record visibility rules, a stage workflow with buttons, audit trail. |

## The data model we're about to write

```
┌─────────────────┐         ┌────────────────────┐         ┌────────────────────┐
│   hcpi.region   │◄────────│ hcpi.field.report  │────────►│ hcpi.field.tag     │
│  (hierarchical) │  1:N    │   (the main one)   │   N:M   │  (lookup labels)   │
└─────────────────┘         └────────────────────┘         └────────────────────┘
                                     │
                                     │ 1:N
                                     ▼
                            ┌────────────────────────┐
                            │ hcpi.field.observation │
                            │  (children of report)  │
                            └────────────────────────┘
```

- **`hcpi.field.report`** — the main entity. One row per day per enumerator.
- **`hcpi.field.observation`** — child rows attached to a report (the issues seen during the day).
- **`hcpi.region`** — a geographical area. Hierarchical (Country → Province → District).
- **`hcpi.field.tag`** — short labels you can pin to a report (Many2many).

Plus we'll extend `res.users` (Odoo's built-in user model) to add a "field reports" smart button.

## Step 1: scaffold the module

We'll use `scaffold` this time (you did the hand-roll exercise in [Your First Module](../../first-edits/your-first-module.md)). Stop Odoo if it's running, then:

```bash
cd /opt/hcpi
source venv/bin/activate
python odoo/odoo-bin scaffold hcpi_field_reports /opt/hcpi/custom/HCPI/
```

You now have `custom/HCPI/hcpi_field_reports/` with the generated skeleton. Open the folder in your editor and **delete what we don't need**:

```bash
cd custom/HCPI/hcpi_field_reports
rm -rf controllers demo
rm views/templates.xml
```

What's left:

```
hcpi_field_reports/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── models.py
├── security/
│   └── ir.model.access.csv
└── views/
    └── views.xml
```

Rename `models/models.py` to something meaningful, and add files for the other models:

```bash
mv models/models.py models/hcpi_field_report.py
touch models/hcpi_field_observation.py
touch models/hcpi_region.py
touch models/hcpi_field_tag.py
touch models/res_users.py
```

Update `models/__init__.py` to import all of them:

```python
from . import hcpi_region
from . import hcpi_field_tag
from . import hcpi_field_observation
from . import hcpi_field_report
from . import res_users
```

**The order matters slightly:** Odoo loads them in this order, and a model that references another (like `hcpi_field_report` referring to `hcpi.region`) can be loaded after the one it references without complaint — but it's cleaner to list "leaf" models first.

Update `__manifest__.py`:

```python
{
    'name': 'HCPI Field Reports',
    'version': '18.0.1.0.0',
    'summary': 'Daily field-activity reports for enumerators and supervisors.',
    'category': 'HCPI',
    'depends': ['base', 'mail'],
    'data': [
        'security/ir.model.access.csv',
        'views/views.xml',
    ],
    'installable': True,
    'application': True,
    'license': 'LGPL-3',
}
```

The new bit: `'mail'` in `depends`. We'll use it in Part 3 for the chatter (comments + audit trail). Add it now so we don't have to migrate the manifest later.

## Step 2: the Region model

Open `models/hcpi_region.py`:

```python
from odoo import api, fields, models


class HcpiRegion(models.Model):
    _name = 'hcpi.region'
    _description = "HCPI Region"
    _parent_name = 'parent_id'
    _parent_store = True
    _rec_name = 'complete_name'
    _order = 'complete_name'

    name = fields.Char(required=True)
    code = fields.Char(size=8)
    parent_id = fields.Many2one(
        'hcpi.region',
        string="Parent Region",
        ondelete='restrict',
        index=True,
    )
    parent_path = fields.Char(index=True, unaccent=False)
    child_ids = fields.One2many('hcpi.region', 'parent_id', string="Sub-regions")
    complete_name = fields.Char(compute='_compute_complete_name', store=True, recursive=True)
    active = fields.Boolean(default=True)

    @api.depends('name', 'parent_id.complete_name')
    def _compute_complete_name(self):
        for region in self:
            if region.parent_id:
                region.complete_name = f"{region.parent_id.complete_name} / {region.name}"
            else:
                region.complete_name = region.name
```

Walking through the new parts:

| Element | What it does |
|---|---|
| `_parent_name = 'parent_id'` + `_parent_store = True` | Tells Odoo: this model is hierarchical. The `parent_path` field will be auto-maintained to enable fast `child_of` / `parent_of` queries. |
| `parent_path` field | Internal Odoo column. You declare it; Odoo populates it. Don't write to it. |
| `child_ids = One2many('hcpi.region', 'parent_id', ...)` | The inverse of `parent_id`. Lets you write `region.child_ids` to get sub-regions. The string `'parent_id'` is the **field on the other side that points back here**. |
| `_rec_name = 'complete_name'` | The field Odoo uses as the "display name" everywhere (dropdowns, breadcrumbs). Default is `name`; here we want `"Uganda / Central / Kampala"` to show up. |
| `recursive=True` on compute | Tells Odoo this compute depends on values from the same model up the chain. Without it Odoo refuses to compute. |
| `active = Boolean(default=True)` | Magic field name — Odoo treats `active=False` as "archived" and hides such records from default searches. |

### Field-type recap with Region

Region uses:

- `Char` (name, code) — short text.
- `Boolean` (active) — true/false.
- `Many2one('hcpi.region', ...)` — foreign key to itself.
- `One2many(..., 'parent_id', ...)` — virtual reverse side of the M2O. No DB column.

## Step 3: the Tag model (Many2many target)

Open `models/hcpi_field_tag.py`:

```python
from odoo import fields, models


class HcpiFieldTag(models.Model):
    _name = 'hcpi.field.tag'
    _description = "Field Report Tag"
    _order = 'name'

    name = fields.Char(required=True)
    color = fields.Integer(string="Color Index", default=0)

    _sql_constraints = [
        ('name_unique', 'unique(name)', "A tag with this name already exists."),
    ]
```

Two new things:

- **`color = Integer(...)`** — Odoo Kanban views use this to colour-code tag chips. The integer is an index into a palette; we'll see it light up in Part 2.
- **`_sql_constraints`** — database-level uniqueness. Tries to enforce at the PostgreSQL level. Format is `(name, sql_fragment, error_message)`.

## Step 4: the Observation model

Open `models/hcpi_field_observation.py`:

```python
from odoo import fields, models


class HcpiFieldObservation(models.Model):
    _name = 'hcpi.field.observation'
    _description = "Field Observation"
    _order = 'severity desc, id'

    name = fields.Char(string="Description", required=True)
    report_id = fields.Many2one(
        'hcpi.field.report',
        string="Report",
        required=True,
        ondelete='cascade',
        index=True,
    )
    outlet_name = fields.Char(string="Outlet")
    severity = fields.Selection(
        [
            ('low', "Low"),
            ('medium', "Medium"),
            ('high', "High"),
        ],
        default='low',
        required=True,
    )
    needs_action = fields.Boolean(default=False)
    notes = fields.Text()
```

Walk through the new field types:

| Field | What it stores |
|---|---|
| `Selection([...])` | A value picked from a fixed list. The CSV is `(database_value, label)`. Renders as a dropdown in the UI. |
| `Text` | Multi-line string. Renders as a `<textarea>`. Use `Char` for short, `Text` for long. |

**`ondelete='cascade'` on the M2O** is critical. It means "when the parent report is deleted, delete me too." Without it, deleting a report with observations would fail with a database integrity error. Alternatives: `'restrict'` (default — blocks the parent delete) and `'set null'` (orphans the child).

## Step 5: the main Field Report model

Open `models/hcpi_field_report.py`:

```python
from odoo import api, fields, models


class HcpiFieldReport(models.Model):
    _name = 'hcpi.field.report'
    _description = "Field Report"
    _order = 'date desc, id desc'

    name = fields.Char(string="Reference", default="New", copy=False, readonly=True)

    date = fields.Date(default=fields.Date.context_today, required=True, index=True)
    enumerator_id = fields.Many2one(
        'res.users',
        string="Enumerator",
        default=lambda self: self.env.user,
        required=True,
        index=True,
    )
    region_id = fields.Many2one('hcpi.region', string="Region", required=True, index=True)

    state = fields.Selection(
        [
            ('draft', "Draft"),
            ('submitted', "Submitted"),
            ('approved', "Approved"),
            ('rejected', "Rejected"),
        ],
        default='draft',
        required=True,
        tracking=True,
    )

    duration_hours = fields.Float(string="Hours in Field", default=0.0)
    outlets_visited = fields.Integer(default=0)
    prices_collected = fields.Integer(default=0)
    notes = fields.Text()

    tag_ids = fields.Many2many('hcpi.field.tag', string="Tags")

    observation_ids = fields.One2many(
        'hcpi.field.observation',
        'report_id',
        string="Observations",
    )

    observation_count = fields.Integer(
        compute='_compute_observation_count',
        store=True,
    )
    has_high_severity = fields.Boolean(
        compute='_compute_has_high_severity',
        store=True,
    )

    @api.depends('observation_ids')
    def _compute_observation_count(self):
        for report in self:
            report.observation_count = len(report.observation_ids)

    @api.depends('observation_ids.severity')
    def _compute_has_high_severity(self):
        for report in self:
            report.has_high_severity = any(o.severity == 'high' for o in report.observation_ids)
```

This is the largest file so far. Reading it as a tour of field types and concepts:

| Pattern | What's new |
|---|---|
| `default="New"` with `readonly=True`, `copy=False` | We'll generate a proper reference (`FR/2026/0001`) in Part 3 with a sequence. For now `"New"` is a placeholder. `copy=False` means it doesn't carry through when a user uses "Duplicate". |
| `default=fields.Date.context_today` | Built-in default for "today, in the user's timezone". |
| `default=lambda self: self.env.user` | The lambda runs at create time and returns the current user. `self.env.user` is how you reach the user inside any model. |
| `index=True` on FKs | Adds a Postgres index on the column. Always do this on M2O fields you'll filter or group by — it's free and makes searches fast. |
| `Float(string="Hours...", default=0.0)` | Decimal number. |
| `Integer(default=0)` | Whole number. |
| `Many2many('hcpi.field.tag')` | Many-to-many through an auto-generated join table. The table name is auto-derived; you can override with `relation=`, `column1=`, `column2=` arguments if you ever need to. |
| `One2many('hcpi.field.observation', 'report_id', ...)` | The reverse side of the M2O in the observation model. `'report_id'` is the M2O field on `hcpi.field.observation` that points here. |
| `compute=...` + `store=True` + `@api.depends(...)` | A stored computed field. Odoo runs the method whenever any field in `@api.depends(...)` changes and writes the result. Stored computes are queryable like normal fields. |
| `tracking=True` on `state` | Tells `mail.thread` to log every change to this field in the chatter. We'll wire up `mail.thread` in Part 3 — adding `tracking=True` now is harmless. |

### Compute methods — the loop pattern

```python
@api.depends('observation_ids.severity')
def _compute_has_high_severity(self):
    for report in self:
        report.has_high_severity = any(o.severity == 'high' for o in report.observation_ids)
```

**Always loop over `self`.** Compute methods are called on a **recordset** (could be 1 record, could be 1000). You don't get to assume length-1.

`@api.depends('observation_ids.severity')` reads "depends on the `severity` field of every record in `observation_ids`". When any observation's severity changes, Odoo re-runs this compute on every report that owns that observation.

### Recursive vs non-recursive computes

`hcpi.region.complete_name` had `recursive=True` because the field depends on its own parent's value (`parent_id.complete_name`). The two computes in `hcpi.field.report` don't depend on themselves, so we don't need that flag.

## Step 6: extending `res.users` with inheritance

Open `models/res_users.py`:

```python
from odoo import api, fields, models


class ResUsers(models.Model):
    _inherit = 'res.users'

    field_report_ids = fields.One2many(
        'hcpi.field.report',
        'enumerator_id',
        string="Field Reports",
    )
    field_report_count = fields.Integer(
        compute='_compute_field_report_count',
    )

    @api.depends('field_report_ids')
    def _compute_field_report_count(self):
        for user in self:
            user.field_report_count = len(user.field_report_ids)
```

This is **`_inherit` with no `_name`** — also called **classical inheritance** in Odoo. It says: "I'm not a new model; I'm modifying `res.users` in place. Add my fields and methods to the existing model."

- The `res_users` PostgreSQL table gets no new columns from the `One2many` (it has no DB column anyway) and no new column from the `Integer` (it's an in-memory compute — `store=False` is the default).
- After install, every `res.users` record has the `field_report_ids` and `field_report_count` attributes available.

There are two other inheritance modes you'll meet in HCPI:

| Mode | Syntax | Effect |
|---|---|---|
| **Classical (extension)** | `_inherit = 'res.users'` (no `_name`) | Adds fields/methods to the existing model. Same table. |
| **Delegation** | `_inherit = 'res.partner'` + `_name = 'my.thing'` | Creates a new model with a hidden FK to the parent. Rare. |
| **Prototype** | `_inherit = ['mail.thread', 'image.mixin']` (no `_name`) on a fresh model | Mixes in features from abstract models. Common — `mail.thread` everywhere. |

We use **classical** in `res_users.py` and we'll use **prototype** in Part 3 when we add `mail.thread` to the report model.

## Step 7: minimal views to see the data

We'll do proper views in Part 2. For now we just need enough UI to create records and confirm the database is wired correctly.

Open `views/views.xml` and replace its content:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <!-- Region: list + form -->
    <record id="view_hcpi_region_list" model="ir.ui.view">
        <field name="name">hcpi.region.list</field>
        <field name="model">hcpi.region</field>
        <field name="arch" type="xml">
            <list>
                <field name="complete_name"/>
                <field name="code"/>
            </list>
        </field>
    </record>

    <record id="view_hcpi_region_form" model="ir.ui.view">
        <field name="name">hcpi.region.form</field>
        <field name="model">hcpi.region</field>
        <field name="arch" type="xml">
            <form>
                <sheet>
                    <group>
                        <field name="name"/>
                        <field name="code"/>
                        <field name="parent_id"/>
                    </group>
                </sheet>
            </form>
        </field>
    </record>

    <record id="action_hcpi_region" model="ir.actions.act_window">
        <field name="name">Regions</field>
        <field name="res_model">hcpi.region</field>
        <field name="view_mode">list,form</field>
    </record>

    <!-- Tag: list + form -->
    <record id="view_hcpi_field_tag_list" model="ir.ui.view">
        <field name="name">hcpi.field.tag.list</field>
        <field name="model">hcpi.field.tag</field>
        <field name="arch" type="xml">
            <list editable="bottom">
                <field name="name"/>
                <field name="color" widget="color_picker"/>
            </list>
        </field>
    </record>

    <record id="action_hcpi_field_tag" model="ir.actions.act_window">
        <field name="name">Tags</field>
        <field name="res_model">hcpi.field.tag</field>
        <field name="view_mode">list,form</field>
    </record>

    <!-- Field Report: list + form -->
    <record id="view_hcpi_field_report_list" model="ir.ui.view">
        <field name="name">hcpi.field.report.list</field>
        <field name="model">hcpi.field.report</field>
        <field name="arch" type="xml">
            <list>
                <field name="name"/>
                <field name="date"/>
                <field name="enumerator_id"/>
                <field name="region_id"/>
                <field name="outlets_visited"/>
                <field name="prices_collected"/>
                <field name="observation_count"/>
                <field name="state"/>
            </list>
        </field>
    </record>

    <record id="view_hcpi_field_report_form" model="ir.ui.view">
        <field name="name">hcpi.field.report.form</field>
        <field name="model">hcpi.field.report</field>
        <field name="arch" type="xml">
            <form>
                <sheet>
                    <group>
                        <group>
                            <field name="date"/>
                            <field name="enumerator_id"/>
                            <field name="region_id"/>
                            <field name="state"/>
                        </group>
                        <group>
                            <field name="duration_hours"/>
                            <field name="outlets_visited"/>
                            <field name="prices_collected"/>
                            <field name="observation_count"/>
                        </group>
                    </group>
                    <field name="tag_ids" widget="many2many_tags" options="{'color_field': 'color'}"/>
                    <notebook>
                        <page string="Observations">
                            <field name="observation_ids">
                                <list editable="bottom">
                                    <field name="name"/>
                                    <field name="outlet_name"/>
                                    <field name="severity"/>
                                    <field name="needs_action"/>
                                </list>
                            </field>
                        </page>
                        <page string="Notes">
                            <field name="notes"/>
                        </page>
                    </notebook>
                </sheet>
            </form>
        </field>
    </record>

    <record id="action_hcpi_field_report" model="ir.actions.act_window">
        <field name="name">Field Reports</field>
        <field name="res_model">hcpi.field.report</field>
        <field name="view_mode">list,form</field>
    </record>

    <!-- Menus -->
    <menuitem id="menu_hcpi_field_reports_root"
              name="Field Reports"
              sequence="50"/>

    <menuitem id="menu_hcpi_field_reports_reports"
              name="Reports"
              parent="menu_hcpi_field_reports_root"
              action="action_hcpi_field_report"
              sequence="10"/>

    <menuitem id="menu_hcpi_field_reports_config"
              name="Configuration"
              parent="menu_hcpi_field_reports_root"
              sequence="100"/>

    <menuitem id="menu_hcpi_regions"
              name="Regions"
              parent="menu_hcpi_field_reports_config"
              action="action_hcpi_region"
              sequence="10"/>

    <menuitem id="menu_hcpi_field_tags"
              name="Tags"
              parent="menu_hcpi_field_reports_config"
              action="action_hcpi_field_tag"
              sequence="20"/>

</odoo>
```

A few things to note:

- **`<list editable="bottom">`** in the tag list — lets you add rows directly in the list view, like a spreadsheet.
- **`widget="color_picker"`** on the color field — Odoo replaces the integer input with a colour palette UI.
- **`<notebook>` and `<page>`** — tabs inside a form. Common pattern for "lots of related data on one form."
- **The embedded `<list>` inside the One2many** — when rendering a One2many, Odoo needs to know which columns to show. The inline `<list>` defines that.
- **`widget="many2many_tags" options="{'color_field': 'color'}"`** — renders the M2M as coloured chips instead of a dropdown.

## Step 8: open up security temporarily

Replace `security/ir.model.access.csv`:

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_hcpi_region,hcpi.region,model_hcpi_region,base.group_user,1,1,1,1
access_hcpi_field_tag,hcpi.field.tag,model_hcpi_field_tag,base.group_user,1,1,1,1
access_hcpi_field_observation,hcpi.field.observation,model_hcpi_field_observation,base.group_user,1,1,1,1
access_hcpi_field_report,hcpi.field.report,model_hcpi_field_report,base.group_user,1,1,1,1
```

Every internal user gets full CRUD on every model. **We'll lock this down properly in Part 3.** For now we want the lowest-friction setup so we can test the data layer.

## Step 9: install and click around

Start Odoo:

```bash
python odoo/odoo-bin -c conf/hcpi.conf
```

In the browser (Developer Mode on):

1. **Apps** → **Update Apps List** → confirm.
2. Search for **HCPI Field Reports**, click **Activate**.
3. The **Field Reports** tile appears in the app launcher.

Open it and:

1. Go to **Configuration → Regions**. Create a tree:
   - Add **Uganda** (no parent).
   - Add **Central** under Uganda.
   - Add **Kampala** under Central.
   - Confirm the **Sub-regions** appear when you open Uganda, and `complete_name` shows **Uganda / Central / Kampala** for the leaf.
2. **Configuration → Tags**. Add `weekend`, `rainy-day`, `mobile-app`. Click colour squares.
3. **Reports → New**. Fill in date, enumerator (yours), region (Kampala), some outlet/price numbers. Add a couple of tags. Switch to the **Observations** tab and add two observations with different severities. Save.

You should see the **Observations count** column populate on the list view (that's `observation_count`, the stored compute).

## Step 10: meet the ORM via the Odoo shell

Open a **second terminal** with the venv activated (leave Odoo running in the first), then:

```bash
cd /opt/hcpi
source venv/bin/activate
python odoo/odoo-bin shell -c conf/hcpi.conf
```

This drops you into a Python REPL with a fully loaded HCPI environment. The variable `self` is bound to `res.users` for the admin user, and `self.env` is the gateway to everything. Try:

```python
# Get a reference to the model (the "manager" — not a record)
Report = env['hcpi.field.report']

# Count all reports
Report.search_count([])

# Find all
all_reports = Report.search([])
len(all_reports)

# Find with a domain
recent = Report.search([('date', '>=', '2026-01-01')])
recent.mapped('name')               # list of references

# Filter by relation
in_kampala = Report.search([('region_id.name', '=', 'Kampala')])

# Read raw values (returns list of dicts)
recent.read(['name', 'date', 'outlets_visited'])

# Browse by id (no SQL query if you just want the recordset)
r = Report.browse(1)
r.name
r.observation_ids                   # the One2many — a recordset
r.observation_ids.mapped('name')

# Create
new = Report.create({
    'date': '2026-05-25',
    'enumerator_id': env.user.id,
    'region_id': env['hcpi.region'].search([('name', '=', 'Kampala')]).id,
    'outlets_visited': 5,
    'prices_collected': 42,
})
new.name        # "New" (until Part 3 wires up the sequence)
new.id

# Create with related records
Report.create({
    'date': '2026-05-25',
    'enumerator_id': env.user.id,
    'region_id': env['hcpi.region'].search([('name', '=', 'Kampala')]).id,
    'observation_ids': [
        (0, 0, {'name': 'Outlet closed', 'severity': 'high'}),
        (0, 0, {'name': 'Price tag missing', 'severity': 'low'}),
    ],
})

# Update one record
new.write({'outlets_visited': 7})

# Update many at once (single UPDATE statement)
recent.write({'state': 'submitted'})

# Delete
new.unlink()

# Persist changes
env.cr.commit()
```

What you've just used:

| Method | What it does | Returns |
|---|---|---|
| `search(domain)` | Find by filter | Recordset |
| `search_count(domain)` | Count | Integer |
| `browse(ids)` | Fetch by known id(s) | Recordset (no SQL yet) |
| `read([fields])` | Pull field values | List of dicts |
| `mapped('field')` | Pull one field from each record | List or recordset |
| `create(values)` | Insert | New recordset |
| `write(values)` | Update — works on any recordset size | `True` |
| `unlink()` | Delete | `True` |

### The "command tuples" for relational writes

That `[(0, 0, {...})]` syntax for `observation_ids` is Odoo's way of saying "create a new observation linked to this report." There are seven such commands:

| Tuple | Meaning |
|---|---|
| `(0, 0, values)` | Create a new linked record |
| `(1, id, values)` | Update existing linked record |
| `(2, id, 0)` | Unlink AND delete |
| `(3, id, 0)` | Unlink (no delete) |
| `(4, id, 0)` | Link existing record |
| `(5, 0, 0)` | Unlink all |
| `(6, 0, ids)` | Replace links with this list |

You'll see all of them in real HCPI code. `(0, 0, ...)` (create) and `(6, 0, ids)` (replace) are the two most common.

## Step 11: domains — the search filter language

Domains are lists of triples (`('field', 'op', 'value')`) combined with logical operators:

```python
# Implicit AND between triples
Report.search([
    ('state', '=', 'approved'),
    ('outlets_visited', '>', 0),
])

# Explicit OR (prefix operators, Polish notation)
Report.search(['|',
    ('state', '=', 'approved'),
    ('state', '=', 'submitted'),
])

# NOT
Report.search(['!', ('state', '=', 'draft')])

# IN
Report.search([('state', 'in', ['approved', 'submitted'])])

# Crossing a relation
Report.search([('region_id.name', '=', 'Kampala')])
Report.search([('observation_ids.severity', '=', 'high')])

# LIKE / ILIKE (case-insensitive)
Report.search([('notes', 'ilike', 'closure')])

# Date comparison
Report.search([('date', '>=', '2026-01-01'), ('date', '<', '2026-04-01')])
```

**Operators you'll use most:** `=`, `!=`, `>`, `>=`, `<`, `<=`, `in`, `not in`, `ilike`, `child_of`, `parent_of`.

Try this last one — it's why we set up `_parent_store`:

```python
# All reports filed in Uganda or any sub-region
Region = env['hcpi.region']
uganda = Region.search([('name', '=', 'Uganda')])
Report.search([('region_id', 'child_of', uganda.id)])
```

`child_of` walks the `parent_path` index — fast even with deep hierarchies.

## Exercises

1. **Compute the average outlets per report per region.** In the shell:

    ```python
    Report = env['hcpi.field.report']
    Region = env['hcpi.region']
    for region in Region.search([]):
        reports = Report.search([('region_id', '=', region.id)])
        if reports:
            avg = sum(r.outlets_visited for r in reports) / len(reports)
            print(f"{region.complete_name}: {avg:.1f}")
    ```

2. **Add a `weekday` computed field** to `hcpi.field.report` — a `Char` that shows `"Monday"`, `"Tuesday"`, etc. derived from `date`. Hint: `fields.Date` returns Python `date` objects; they have `.strftime('%A')`. The compute will need `@api.depends('date')` and shouldn't be stored (cheap, not searched).

3. **Add an `_sql_constraints`** on the report saying "one report per enumerator per date" (unique on `(enumerator_id, date)`). Hint: same pattern as the tag constraint, multi-column tuple in SQL.

4. **Find all reports tagged "weekend".** Write a domain. Hint: cross the M2M with `('tag_ids.name', '=', 'weekend')`.

5. **Bulk update.** Set `state = 'submitted'` for every draft report dated before today. One ORM call.

## What you learned

After Part 1 you understand:

- **Field types**: `Char`, `Text`, `Integer`, `Float`, `Boolean`, `Date`, `Selection`.
- **Relationships**: `Many2one` (with `ondelete=`), `One2many` (with the inverse field name), `Many2many` (with auto join tables).
- **Hierarchical models**: `_parent_name`, `_parent_store`, `parent_path`, and `child_of` queries.
- **Computed fields**: stored vs unstored, `@api.depends`, the for-loop-over-self pattern.
- **Inheritance** (classical): adding fields to `res.users` without making a new model.
- **`_sql_constraints`** for database-level uniqueness.
- **The ORM**: `search`, `browse`, `read`, `create`, `write`, `unlink`, `mapped`, and command tuples for relational writes.
- **Domains**: AND/OR/NOT, cross-relation lookups, `child_of`.

## What's next

➡️ **[Part 2: Views & UX](part2-views.md)** — turn this raw module into something pleasant to use. Statusbar workflow, Kanban with stages, Graph and Pivot reports, Calendar, advanced search, inline editing, list decorations, and a printable PDF.
