# Building an HCPI Module — Part 1: Models, Fields, Relationships

<!--
Duration: half a day (about 4 hours including breaks).
Covers: module structure (manifest, package layout), models, field types,
relationships (Many2one, One2many, Many2many, hierarchical), computed fields,
model inheritance, the ORM, minimal views to display the data.
Prereq: you've finished the Setup pages and can reach http://localhost:9201.
       Python intro recommended but not required.
-->

This is the start of a three-part tutorial. You'll build **`hcpi_outlet_onboarding`** — a small but realistic HCPI module the statistical office uses **back at the desk** to propose, inspect, and approve new outlets before they go onto the price-collection list.

We choose this scenario because the real price-capture work happens on **tablets, in the field**, through the HCPI Flutter app — not in the web UI. The web is where supervisors, coordinators, and data analysts do their back-office work, and outlet onboarding is exactly that kind of work: someone hears about a new market, proposes it, a junior staffer inspects it, a manager approves it, and only *then* does the outlet appear in the collection rotation.

By the end of all three parts you'll have touched every basic concept in the HCPI back-end programme: models, every common field type and relationship, the ORM, all view types, search, security, workflow, and chatter.

## What we're building, across all three parts

The story:

1. A supervisor hears about a new shop or market — call it a candidate outlet.
2. They create an **Outlet Proposal** in HCPI with the name, address, contact details, GPS coordinates, and proposed type (supermarket / open market / pharmacy / etc.).
3. A field officer is sent out to **inspect** the location. They log the visit: did they find it, is it operating, are prices visible? They can attach a photo.
4. After enough inspections, the proposal goes **under review**. A manager either **approves** (the outlet will be added to the collection list) or **rejects**.

What each part builds:

| Part | Focus | What you end up with |
|---|---|---|
| **1 (this page)** | Module scaffolding, models, fields, relationships, ORM | A working module with four models, list and form views, data you can create and query. |
| **[2: Views & UX](part2-views.md)** | All view types, search, reports | Kanban with stages, graphs, pivot, calendar, filters, a printable dossier PDF. |
| **[3: Security & Polish](part3-security.md)** | Groups, access rules, workflow, chatter | Production-ready: roles, record visibility rules, a stage workflow with buttons, audit trail. |

## The data model we're about to write

```
┌─────────────────┐         ┌──────────────────────┐         ┌────────────────────┐
│   hcpi.region   │◄────────│ hcpi.outlet.proposal │────────►│ hcpi.outlet.tag    │
│  (hierarchical) │  N:1    │     (the main one)   │   N:M   │  (lookup labels)   │
└─────────────────┘         └──────────────────────┘         └────────────────────┘
                                       │  ▲
                                       │  │
                                  ┌────▼──┴─────────┐         ┌────────────────────┐
                                  │ hcpi.outlet.    │         │ hcpi.outlet.type   │
                                  │ inspection      │         │ (lookup)           │
                                  │ (child rows)    │◄────────│                    │
                                  └─────────────────┘   N:1   └────────────────────┘
                                                ▲
                                                │ N:1
                                                │
                                          (also linked to type on the proposal)
```

- **`hcpi.outlet.proposal`** — the main entity. One row per candidate outlet.
- **`hcpi.outlet.inspection`** — child rows attached to a proposal (one or more inspection visits).
- **`hcpi.region`** — a geographical area. Hierarchical (Country → Province → District).
- **`hcpi.outlet.type`** — pre-defined outlet categories (Supermarket, Open Market, Pharmacy, …).
- **`hcpi.outlet.tag`** — short freeform labels you can pin to a proposal (Many2many).

We'll also extend `res.users` (Odoo's built-in user model) to add a smart button counting how many proposals each user has filed.

## What's in a module — a quick refresher

An Odoo module is just a folder under `addons_path` (configured in [`hcpi.conf`](../day1/configuration.md)) containing a few specific files. The skeleton:

```
hcpi_outlet_onboarding/
├── __init__.py            ← Python package marker; loads sub-packages
├── __manifest__.py        ← Odoo's metadata: name, dependencies, data files
├── models/                ← Python: classes that become database tables
├── security/              ← Who can do what
├── views/                 ← XML: how data is displayed in the browser
└── data/                  ← XML: seed data, sequences, etc. (used in Part 3)
```

The build path for any feature is the same: **add a model in `models/`, add views to display it in `views/`, declare access in `security/`, list the new XML files in `__manifest__.py`, restart Odoo with `-u hcpi_outlet_onboarding`.**

## Step 1: scaffold the module

Odoo ships a generator that creates the folder skeleton for you. Stop Odoo if it's running, then in a terminal with the venv activated:

```bash
cd /opt/hcpi
source venv/bin/activate
python odoo/odoo-bin scaffold hcpi_outlet_onboarding /opt/hcpi/custom/HCPI/
```

You now have `custom/HCPI/hcpi_outlet_onboarding/` with a generated skeleton. Open the folder in VS Code (`code .` from inside it on WSL).

The generator creates a bunch of things you don't need yet. Delete them so the layout's clean:

```bash
cd /opt/hcpi/custom/HCPI/hcpi_outlet_onboarding
rm -rf controllers demo
rm views/templates.xml
```

What you're left with:

```
hcpi_outlet_onboarding/
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

We'll fill out one file per model. Rename and add:

```bash
rm models/models.py
touch models/hcpi_region.py
touch models/hcpi_outlet_type.py
touch models/hcpi_outlet_tag.py
touch models/hcpi_outlet_inspection.py
touch models/hcpi_outlet_proposal.py
touch models/res_users.py
```

## Step 2: wire up `__init__.py` and the manifest

Open `models/__init__.py` and list every model file:

```python
from . import hcpi_region
from . import hcpi_outlet_type
from . import hcpi_outlet_tag
from . import hcpi_outlet_inspection
from . import hcpi_outlet_proposal
from . import res_users
```

This file is plain Python. Each line tells Python "also import this sub-module when the package loads." If you forget a line, the corresponding model **silently won't load** — no error, just a missing model. This is the most common newcomer bug, so check this file first when something "doesn't exist."

The outer `__init__.py` (at the module root) already has `from . import models` from the scaffold — leave it.

Now `__manifest__.py`:

```python
{
    'name': 'HCPI Outlet Onboarding',
    'version': '18.0.1.0.0',
    'summary': 'Propose, inspect, and approve candidate outlets before they join the price-collection list.',
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

What each key means:

| Key | What it does |
|---|---|
| `name` | Human label shown in the Apps screen. |
| `version` | Convention: `<odoo-major>.<your-major>.<minor>.<patch>` — here `18.0.1.0.0`. |
| `summary` | One-line description on the Apps tile. |
| `category` | Group in the Apps screen. Free-form. |
| `depends` | Other modules required before this one can install. `base` is Odoo's foundation; `mail` gives us the chatter/audit trail we'll wire up in Part 3. |
| `data` | XML/CSV files Odoo loads on install. **Order matters** — security first, then views. |
| `installable` | `False` makes Odoo silently skip the module. |
| `application` | `True` makes it appear as a top-level app in the launcher. |
| `license` | Required in modern Odoo. `LGPL-3` is the safe default. |

Notice **models aren't listed in `data`**. Models are Python — they load automatically through the `__init__.py` chain. Only XML/CSV files need to be enumerated.

## Step 3: the Region model

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

Walking through line by line:

| Element | What it does |
|---|---|
| `_name = 'hcpi.region'` | The technical name of the model. The PostgreSQL table will be `hcpi_region` (dots become underscores). |
| `_description` | Human label. Required in modern Odoo; you get a startup warning if it's missing. |
| `_parent_name = 'parent_id'` + `_parent_store = True` | Tells Odoo: this model is hierarchical. The `parent_path` field will be auto-maintained to enable fast `child_of` queries. |
| `parent_path` field | Internal Odoo column. You declare it; Odoo populates it. Don't write to it. |
| `_rec_name = 'complete_name'` | The field Odoo uses as the "display name" everywhere (dropdowns, breadcrumbs). Default is `name`; here we want the full breadcrumb like `"Uganda / Central / Kampala"`. |
| `_order = 'complete_name'` | Default sort order in lists. |
| `Char` (name, code) | Short text — VARCHAR. `size=8` caps it; without `size` it's unlimited. |
| `Boolean` (active) | True/false. `active` is a special name — Odoo treats records with `active=False` as archived and hides them from default searches. |
| `Many2one('hcpi.region', ...)` | A foreign key to another model (here, itself, because it's hierarchical). |
| `ondelete='restrict'` | What happens if the parent gets deleted. `restrict` blocks the delete; `cascade` deletes children too; `set null` orphans them. |
| `One2many('hcpi.region', 'parent_id', ...)` | The reverse side of the M2O. Has no DB column — it's computed by looking at the other side. The `'parent_id'` is the field on the *other* model that points back here. |
| `compute='_compute_complete_name', store=True, recursive=True` | A *computed field*. `store=True` writes the result to the database (so you can search by it). `recursive=True` is needed because the compute depends on its own parent's value. |
| `@api.depends('name', 'parent_id.complete_name')` | Tells Odoo "re-run this compute whenever these fields change." |

### About the compute method

```python
@api.depends('name', 'parent_id.complete_name')
def _compute_complete_name(self):
    for region in self:
        ...
```

**Always loop over `self`.** A compute is called on a *recordset* — could be one record, could be a thousand. You don't get to assume length-1.

## Step 4: the OutletType model

Open `models/hcpi_outlet_type.py`:

```python
from odoo import fields, models


class HcpiOutletType(models.Model):
    _name = 'hcpi.outlet.type'
    _description = "Outlet Type"
    _order = 'name'

    name = fields.Char(required=True)
    code = fields.Char(size=8)
    description = fields.Text()
    active = fields.Boolean(default=True)

    _sql_constraints = [
        ('code_unique', 'unique(code)', "An outlet type with this code already exists."),
    ]
```

Two new things:

- **`Text`** (description) — multi-line string. Renders as a textarea. Use `Char` for short, `Text` for long.
- **`_sql_constraints`** — database-level constraints, enforced by PostgreSQL itself. Faster and more reliable than checking in Python because the database can't be bypassed. Format is `(name, sql_fragment, error_message)`.

## Step 5: the Tag model (Many2many target)

Open `models/hcpi_outlet_tag.py`:

```python
from odoo import fields, models


class HcpiOutletTag(models.Model):
    _name = 'hcpi.outlet.tag'
    _description = "Outlet Tag"
    _order = 'name'

    name = fields.Char(required=True)
    color = fields.Integer(string="Color Index", default=0)

    _sql_constraints = [
        ('name_unique', 'unique(name)', "A tag with this name already exists."),
    ]
```

- **`Integer`** (color) — Odoo Kanban views use this as an index into a colour palette. We'll light it up in Part 2.

## Step 6: the Inspection model (a child)

Open `models/hcpi_outlet_inspection.py`:

```python
from odoo import fields, models


class HcpiOutletInspection(models.Model):
    _name = 'hcpi.outlet.inspection'
    _description = "Outlet Inspection"
    _order = 'date desc, id desc'

    name = fields.Char(string="Summary", required=True)
    proposal_id = fields.Many2one(
        'hcpi.outlet.proposal',
        string="Proposal",
        required=True,
        ondelete='cascade',
        index=True,
    )
    date = fields.Date(default=fields.Date.context_today, required=True)
    inspector_id = fields.Many2one(
        'res.users',
        string="Inspector",
        default=lambda self: self.env.user,
        required=True,
    )
    result = fields.Selection(
        [
            ('pass', "Pass"),
            ('partial', "Partial"),
            ('fail', "Fail"),
        ],
        default='partial',
        required=True,
    )
    photo = fields.Binary(attachment=True)
    observations = fields.Text()
```

New things to notice:

| Field/feature | What it does |
|---|---|
| `Date` | Date without time. `fields.Date.context_today` is "today in the user's timezone." |
| `Selection([...])` | A value picked from a fixed list. The tuples are `(database_value, label)`. Renders as a dropdown. |
| `Binary(attachment=True)` | A file (photo, PDF, anything). `attachment=True` stores it outside the table to keep queries fast. |
| `ondelete='cascade'` on the M2O to the proposal | "When the parent proposal is deleted, delete me too." Without this, deleting a proposal that has inspections fails with a database integrity error. |
| `default=lambda self: self.env.user` | A function that runs at create-time and returns the current user. `self.env.user` is how you reach the current user from inside any model. |

## Step 7: the main Outlet Proposal model

Open `models/hcpi_outlet_proposal.py`:

```python
from odoo import api, fields, models


class HcpiOutletProposal(models.Model):
    _name = 'hcpi.outlet.proposal'
    _description = "Outlet Proposal"
    _order = 'create_date desc'

    # Identity
    name = fields.Char(string="Reference", default="New", copy=False, readonly=True)
    outlet_name = fields.Char(string="Outlet Name", required=True)

    # Workflow
    state = fields.Selection(
        [
            ('draft', "Draft"),
            ('inspecting', "Inspecting"),
            ('review', "Under Review"),
            ('approved', "Approved"),
            ('rejected', "Rejected"),
        ],
        default='draft',
        required=True,
        tracking=True,
    )

    # Classification
    region_id = fields.Many2one('hcpi.region', string="Region", required=True, index=True)
    outlet_type_id = fields.Many2one('hcpi.outlet.type', string="Outlet Type", required=True)
    tag_ids = fields.Many2many('hcpi.outlet.tag', string="Tags")

    # Location / contact
    address = fields.Text()
    latitude = fields.Float(digits=(10, 6))
    longitude = fields.Float(digits=(10, 6))
    contact_name = fields.Char()
    contact_phone = fields.Char()
    operating_hours = fields.Char(help="e.g. Mon–Sat, 7am–7pm")

    # People
    proposed_by = fields.Many2one(
        'res.users',
        string="Proposed By",
        default=lambda self: self.env.user,
        required=True,
        index=True,
    )
    reviewed_by = fields.Many2one('res.users', string="Reviewed By", readonly=True)
    approval_date = fields.Date(readonly=True)

    # Notes
    notes = fields.Text()

    # Inspections (children)
    inspection_ids = fields.One2many(
        'hcpi.outlet.inspection',
        'proposal_id',
        string="Inspections",
    )

    # Computed
    inspection_count = fields.Integer(
        compute='_compute_inspection_stats',
        store=True,
    )
    last_inspection_date = fields.Date(
        compute='_compute_inspection_stats',
        store=True,
    )
    has_failed_inspection = fields.Boolean(
        compute='_compute_has_failed_inspection',
        store=True,
    )

    @api.depends('inspection_ids', 'inspection_ids.date')
    def _compute_inspection_stats(self):
        for proposal in self:
            proposal.inspection_count = len(proposal.inspection_ids)
            dates = proposal.inspection_ids.mapped('date')
            proposal.last_inspection_date = max(dates) if dates else False

    @api.depends('inspection_ids.result')
    def _compute_has_failed_inspection(self):
        for proposal in self:
            proposal.has_failed_inspection = any(
                i.result == 'fail' for i in proposal.inspection_ids
            )
```

This is the largest file so far. The new patterns:

| Pattern | What's new |
|---|---|
| `default="New"` + `readonly=True` + `copy=False` | `name` is a reference like `OP/2026/0001` — we'll generate it from a sequence in Part 3. For now `"New"` is a placeholder. `copy=False` means the value is *not* carried through when a user duplicates a record. |
| `tracking=True` on `state` | Tells `mail.thread` to log every change in the chatter. We'll wire up `mail.thread` in Part 3 — `tracking=True` here is harmless until then. |
| `Float(digits=(10, 6))` | Decimal number. `digits=(precision, scale)` means up to 10 digits with 6 after the decimal point — perfect for GPS coordinates. |
| `Many2many('hcpi.outlet.tag')` | Many-to-many via an auto-generated PostgreSQL join table. You can override the relation name and column names with `relation=`, `column1=`, `column2=` arguments, but defaults are fine 99% of the time. |
| `One2many(..., 'proposal_id', ...)` | The reverse side of the M2O on the inspection model. `'proposal_id'` is the field on `hcpi.outlet.inspection` that points back here. |
| `compute=...` + `store=True` + `@api.depends(...)` | Stored computed fields. Odoo runs the method whenever any field listed in `@api.depends(...)` changes, and writes the result to the table. Because they're stored, you can search and group by them. |
| `inspection_ids.mapped('date')` | `mapped('field')` is a recordset method that returns a list of that field's values across all records — like a list comprehension `[i.date for i in inspection_ids]` but more concise. |
| `max(dates) if dates else False` | A *ternary expression*. Equivalent to `if dates: ... else: ...`. Odoo uses `False` instead of `None` for empty Date fields. |

### Two computes, one method

Notice `inspection_count` and `last_inspection_date` both reference the same compute method `_compute_inspection_stats`. That's allowed and idiomatic when two fields are calculated together — saves looping twice over `self`. As long as both `@api.depends` triggers are listed once, Odoo handles re-computing.

## Step 8: extending `res.users` with inheritance

Open `models/res_users.py`:

```python
from odoo import api, fields, models


class ResUsers(models.Model):
    _inherit = 'res.users'

    proposal_ids = fields.One2many(
        'hcpi.outlet.proposal',
        'proposed_by',
        string="Outlet Proposals",
    )
    proposal_count = fields.Integer(
        compute='_compute_proposal_count',
    )

    @api.depends('proposal_ids')
    def _compute_proposal_count(self):
        for user in self:
            user.proposal_count = len(user.proposal_ids)
```

This is **`_inherit` with no `_name`** — also called **classical inheritance** in Odoo. It says: "I'm not a new model; I'm modifying `res.users` in place. Add my fields and methods to the existing model."

- The `res_users` PostgreSQL table gets no new columns from the `One2many` (it has no DB column anyway) and no new column from the `Integer` (it's an in-memory compute — `store=False` is the default).
- After install, every `res.users` record has `proposal_ids` and `proposal_count` available — so you can read `env.user.proposal_count` from anywhere.

There are three inheritance modes in Odoo overall:

| Mode | Syntax | Effect |
|---|---|---|
| **Classical (extension)** | `_inherit = 'res.users'` (no `_name`) | Adds fields/methods to the existing model. Same table. |
| **Delegation** | `_inherits = {'res.partner': 'partner_id'}` (note the `s`) | Creates a new model with a hidden FK to the parent. Rare. |
| **Prototype** | `_inherit = ['mail.thread']` on a fresh model that has its own `_name` | Mixes in features from abstract models. Used everywhere — `mail.thread` for chatter, etc. |

We use **classical** in `res_users.py`, and we'll use **prototype** in Part 3 when we add `mail.thread` to the proposal model.

## Step 9: minimal views to see the data

Real views come in Part 2 — for now we just need enough UI to create records and confirm the database is wired correctly.

Open `views/views.xml` and replace its content:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <!-- ============ Region ============ -->
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

    <!-- ============ Outlet Type ============ -->
    <record id="view_hcpi_outlet_type_list" model="ir.ui.view">
        <field name="name">hcpi.outlet.type.list</field>
        <field name="model">hcpi.outlet.type</field>
        <field name="arch" type="xml">
            <list editable="bottom">
                <field name="name"/>
                <field name="code"/>
                <field name="description"/>
            </list>
        </field>
    </record>

    <record id="action_hcpi_outlet_type" model="ir.actions.act_window">
        <field name="name">Outlet Types</field>
        <field name="res_model">hcpi.outlet.type</field>
        <field name="view_mode">list,form</field>
    </record>

    <!-- ============ Tag ============ -->
    <record id="view_hcpi_outlet_tag_list" model="ir.ui.view">
        <field name="name">hcpi.outlet.tag.list</field>
        <field name="model">hcpi.outlet.tag</field>
        <field name="arch" type="xml">
            <list editable="bottom">
                <field name="name"/>
                <field name="color" widget="color_picker"/>
            </list>
        </field>
    </record>

    <record id="action_hcpi_outlet_tag" model="ir.actions.act_window">
        <field name="name">Tags</field>
        <field name="res_model">hcpi.outlet.tag</field>
        <field name="view_mode">list,form</field>
    </record>

    <!-- ============ Outlet Proposal ============ -->
    <record id="view_hcpi_outlet_proposal_list" model="ir.ui.view">
        <field name="name">hcpi.outlet.proposal.list</field>
        <field name="model">hcpi.outlet.proposal</field>
        <field name="arch" type="xml">
            <list>
                <field name="name"/>
                <field name="outlet_name"/>
                <field name="region_id"/>
                <field name="outlet_type_id"/>
                <field name="proposed_by"/>
                <field name="inspection_count"/>
                <field name="state"/>
            </list>
        </field>
    </record>

    <record id="view_hcpi_outlet_proposal_form" model="ir.ui.view">
        <field name="name">hcpi.outlet.proposal.form</field>
        <field name="model">hcpi.outlet.proposal</field>
        <field name="arch" type="xml">
            <form>
                <sheet>
                    <div class="oe_title">
                        <label for="outlet_name"/>
                        <h1><field name="outlet_name" placeholder="e.g. Kikuubo Wholesale Market"/></h1>
                    </div>
                    <group>
                        <group>
                            <field name="name"/>
                            <field name="state"/>
                            <field name="region_id"/>
                            <field name="outlet_type_id"/>
                            <field name="proposed_by"/>
                        </group>
                        <group>
                            <field name="contact_name"/>
                            <field name="contact_phone"/>
                            <field name="operating_hours"/>
                            <field name="latitude"/>
                            <field name="longitude"/>
                            <field name="inspection_count"/>
                        </group>
                    </group>
                    <field name="tag_ids" widget="many2many_tags" options="{'color_field': 'color'}"/>
                    <notebook>
                        <page string="Address">
                            <field name="address" placeholder="Street, landmark, neighbourhood..."/>
                        </page>
                        <page string="Inspections">
                            <field name="inspection_ids">
                                <list editable="bottom">
                                    <field name="date"/>
                                    <field name="name"/>
                                    <field name="inspector_id"/>
                                    <field name="result"/>
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

    <record id="action_hcpi_outlet_proposal" model="ir.actions.act_window">
        <field name="name">Outlet Proposals</field>
        <field name="res_model">hcpi.outlet.proposal</field>
        <field name="view_mode">list,form</field>
    </record>

    <!-- ============ Menus ============ -->
    <menuitem id="menu_hcpi_outlet_onboarding_root"
              name="Outlet Onboarding"
              sequence="40"/>

    <menuitem id="menu_hcpi_outlet_proposals"
              name="Proposals"
              parent="menu_hcpi_outlet_onboarding_root"
              action="action_hcpi_outlet_proposal"
              sequence="10"/>

    <menuitem id="menu_hcpi_outlet_onboarding_config"
              name="Configuration"
              parent="menu_hcpi_outlet_onboarding_root"
              sequence="100"/>

    <menuitem id="menu_hcpi_regions"
              name="Regions"
              parent="menu_hcpi_outlet_onboarding_config"
              action="action_hcpi_region"
              sequence="10"/>

    <menuitem id="menu_hcpi_outlet_types"
              name="Outlet Types"
              parent="menu_hcpi_outlet_onboarding_config"
              action="action_hcpi_outlet_type"
              sequence="20"/>

    <menuitem id="menu_hcpi_outlet_tags"
              name="Tags"
              parent="menu_hcpi_outlet_onboarding_config"
              action="action_hcpi_outlet_tag"
              sequence="30"/>

</odoo>
```

A few details worth flagging:

- **`<list editable="bottom">`** in the tag and outlet-type lists — lets you add rows directly in the list view, like a spreadsheet. Saves opening a form for each one.
- **`widget="color_picker"`** on the colour field — Odoo replaces the integer input with a colour palette UI.
- **`<notebook>` and `<page>`** — tabs inside a form. Common pattern for "lots of related data on one form."
- **The inline `<list>` inside the One2many** — when rendering a One2many, Odoo needs to know which columns to show. The inline `<list>` defines that.
- **`widget="many2many_tags" options="{'color_field': 'color'}"`** — renders the M2M as coloured chips instead of a dropdown.
- **Menus are records** — Odoo stores them in `ir.ui.menu`. `<menuitem>` is shorthand for creating that record.

## Step 10: open up security temporarily

Replace `security/ir.model.access.csv`:

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_hcpi_region,hcpi.region,model_hcpi_region,base.group_user,1,1,1,1
access_hcpi_outlet_type,hcpi.outlet.type,model_hcpi_outlet_type,base.group_user,1,1,1,1
access_hcpi_outlet_tag,hcpi.outlet.tag,model_hcpi_outlet_tag,base.group_user,1,1,1,1
access_hcpi_outlet_inspection,hcpi.outlet.inspection,model_hcpi_outlet_inspection,base.group_user,1,1,1,1
access_hcpi_outlet_proposal,hcpi.outlet.proposal,model_hcpi_outlet_proposal,base.group_user,1,1,1,1
```

What's going on here:

- The file is a **CSV** because Odoo's `ir.model.access` records are simple enough that a CSV is easier to read than equivalent XML.
- One row per (model, group) pair.
- **`model_<model_table>`** is the auto-generated XML ID for each model's `ir.model` record. Pattern: take the model's table name (`hcpi.region` → `hcpi_region`) and prepend `model_`.
- **`base.group_user`** is the built-in group "every internal user."
- **`1,1,1,1`** = read, write, create, unlink (i.e. delete).

Every internal user gets full CRUD on every model. **We'll lock this down properly in Part 3** with multi-group rules and record-level restrictions. For now we want the lowest-friction setup so we can test the data layer.

## Step 11: install and click around

Start Odoo:

```bash
python odoo/odoo-bin -c conf/hcpi.conf
```

In the browser (developer mode on — see [User Administration](../day1/user-administration.md#activating-developer-mode)):

1. **Apps** → **Update Apps List** → confirm.
2. Search for **HCPI Outlet Onboarding** (remove the default "Apps" filter to see uninstalled modules).
3. Click **Activate** on the tile.
4. The **Outlet Onboarding** main menu appears.

Open it and:

1. **Configuration → Regions**. Build a small tree:
    - Add **Uganda** (no parent).
    - Add **Central** under Uganda.
    - Add **Kampala** under Central.
    - When you open Kampala you should see `complete_name = "Uganda / Central / Kampala"` — that's the recursive compute at work.
2. **Configuration → Outlet Types**. Add a few rows directly in the list: Supermarket (`SUP`), Open Market (`OM`), Pharmacy (`PH`), Convenience Store (`CS`).
3. **Configuration → Tags**. Add `weekend-only`, `large-volume`, `revisit-needed`. Click each colour square to set a colour.
4. **Proposals → New**. Fill in:
    - Outlet name: *Kikuubo Wholesale Market*
    - Region: *Uganda / Central / Kampala*
    - Outlet type: *Open Market*
    - Contact + GPS + operating hours
    - Add a tag or two
    - Switch to the **Inspections** tab and add two inspection rows with different results
    - Save.

You should see the **Inspections** count column populate on the list view (that's `inspection_count`, the stored compute). Edit one of the inspection rows to change its `result` to `fail` and save — `has_failed_inspection` flips on the proposal.

## Step 12: meet the ORM via the Odoo shell

Open a **second terminal** with the venv activated (leave Odoo running in the first), then:

```bash
cd /opt/hcpi
source venv/bin/activate
python odoo/odoo-bin shell -c conf/hcpi.conf
```

This drops you into a Python REPL with a fully loaded HCPI environment. The variable `env` is your gateway to everything.

The ORM is the layer that lets you read and write database records without writing SQL. You use it constantly — in compute methods, in `create()` overrides, in button handlers, everywhere.

Try:

```python
# Get a reference to a model (the "manager" — not a record)
Proposal = env['hcpi.outlet.proposal']

# Count all proposals
Proposal.search_count([])

# Find all (empty domain matches everything)
all_proposals = Proposal.search([])
len(all_proposals)

# Find with a filter (domain)
recent = Proposal.search([('create_date', '>=', '2026-01-01')])
recent.mapped('outlet_name')        # list of names

# Filter by relation (cross-table)
kampala_proposals = Proposal.search([('region_id.name', '=', 'Kampala')])

# Read raw values
recent.read(['name', 'outlet_name', 'state'])

# Browse by id (no SQL query yet — Odoo just gives you the recordset)
p = Proposal.browse(1)
p.outlet_name
p.inspection_ids                    # the One2many — a recordset
p.inspection_ids.mapped('name')

# Create
new = Proposal.create({
    'outlet_name': 'Owino Market Stall 14',
    'region_id': env['hcpi.region'].search([('name', '=', 'Kampala')]).id,
    'outlet_type_id': env['hcpi.outlet.type'].search([('code', '=', 'OM')]).id,
})
new.name        # "New" — until Part 3 wires up the sequence

# Create with related children — using a "command tuple"
Proposal.create({
    'outlet_name': 'Nakasero Pharmacy',
    'region_id': env['hcpi.region'].search([('name', '=', 'Kampala')]).id,
    'outlet_type_id': env['hcpi.outlet.type'].search([('code', '=', 'PH')]).id,
    'inspection_ids': [
        (0, 0, {'name': 'First visit', 'result': 'partial'}),
        (0, 0, {'name': 'Follow-up', 'result': 'pass'}),
    ],
})

# Update one record
new.write({'state': 'inspecting'})

# Update many at once — one UPDATE statement under the hood
recent.write({'state': 'inspecting'})

# Delete
new.unlink()

# Persist changes from the shell
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

### Command tuples for relational writes

That `[(0, 0, {...})]` syntax is Odoo's way of saying "create a new linked record." There are seven such commands:

| Tuple | Meaning |
|---|---|
| `(0, 0, values)` | Create a new linked record |
| `(1, id, values)` | Update existing linked record |
| `(2, id, 0)` | Unlink AND delete |
| `(3, id, 0)` | Unlink (don't delete) |
| `(4, id, 0)` | Link an existing record |
| `(5, 0, 0)` | Unlink all |
| `(6, 0, ids)` | Replace links with this list |

You'll see all of them in real HCPI code. `(0, 0, ...)` (create) and `(6, 0, ids)` (replace) are the two most common.

## Step 13: domains — the search filter language

Domains are lists of triples (`('field', 'op', 'value')`) combined with logical operators. They're how you say "give me records matching this pattern":

```python
# Implicit AND between triples
Proposal.search([
    ('state', '=', 'inspecting'),
    ('outlet_type_id.code', '=', 'OM'),
])

# Explicit OR (prefix operators, Polish notation)
Proposal.search(['|',
    ('state', '=', 'approved'),
    ('state', '=', 'review'),
])

# NOT
Proposal.search(['!', ('state', '=', 'draft')])

# IN
Proposal.search([('state', 'in', ['approved', 'review'])])

# Crossing a relation (dot notation)
Proposal.search([('region_id.name', '=', 'Kampala')])
Proposal.search([('inspection_ids.result', '=', 'fail')])

# LIKE / ILIKE (case-insensitive partial match)
Proposal.search([('outlet_name', 'ilike', 'market')])

# Date comparison
Proposal.search([('create_date', '>=', '2026-01-01')])
```

**Operators you'll use most:** `=`, `!=`, `>`, `>=`, `<`, `<=`, `in`, `not in`, `ilike`, `child_of`, `parent_of`.

Try `child_of` — it's why we set up `_parent_store` on the Region model:

```python
# All proposals in Uganda or any of its sub-regions
Region = env['hcpi.region']
uganda = Region.search([('name', '=', 'Uganda')])
Proposal.search([('region_id', 'child_of', uganda.id)])
```

`child_of` walks the `parent_path` index — fast even with deep hierarchies. Useful when the user picks "Uganda" and you want all proposals in *any* Ugandan region.

## Exercises

1. **Average inspections per outlet type.** In the shell, loop over all outlet types and print the average number of inspections per proposal of that type. Hint: `sum([...]) / len([...])` with a list comprehension.

2. **Add a `weekday` computed field** to `hcpi.outlet.inspection` — a `Char` that shows `"Monday"`, `"Tuesday"`, etc. derived from `date`. Hint: `fields.Date` values are Python `date` objects; they have `.strftime('%A')`. The compute will need `@api.depends('date')` and doesn't need to be stored.

3. **`_sql_constraints` for uniqueness.** Add a constraint on `hcpi.outlet.proposal` saying "one proposal per outlet name per region" (unique on `(outlet_name, region_id)`). Hint: same shape as the constraints in `hcpi.outlet.type` and `hcpi.outlet.tag`, but with a comma-separated column list.

4. **Find all proposals tagged "large-volume".** Write a domain. Hint: cross the M2M with `('tag_ids.name', '=', 'large-volume')`.

5. **Bulk update.** Set `state = 'inspecting'` for every draft proposal that already has at least one inspection. One ORM call.

## What you learned

After Part 1 you've covered the entire data layer of an HCPI module:

- **Module scaffolding**: manifest, `__init__.py` package chain, `data` files, `depends`.
- **Field types**: `Char`, `Text`, `Integer`, `Float` (with `digits=`), `Boolean`, `Date`, `Selection`, `Binary`.
- **Relationships**: `Many2one` (with `ondelete=`), `One2many` (with the inverse field name), `Many2many` (with auto join tables).
- **Hierarchical models**: `_parent_name`, `_parent_store`, `parent_path`, and `child_of` queries.
- **Computed fields**: stored vs unstored, `@api.depends`, the for-loop-over-self pattern, two fields sharing one compute method.
- **Inheritance** (classical): adding fields to `res.users` without making a new model.
- **`_sql_constraints`** for database-level uniqueness.
- **Minimal views**: list, form, notebook + page, inline One2many, menus.
- **CSV-format access control** — open today, locked down in Part 3.
- **The ORM**: `search`, `browse`, `read`, `create`, `write`, `unlink`, `mapped`, and command tuples.
- **Domains**: AND/OR/NOT, cross-relation lookups via dot notation, `child_of`.

## What's next

➡️ **[Part 2: Views & UX](part2-views.md)** — turn this raw module into something pleasant to use. Statusbar workflow, Kanban with stages, Graph and Pivot reports, Calendar, advanced search, list decorations, and a printable PDF dossier.
