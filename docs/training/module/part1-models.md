# Building an HCPI Module — Part 1: Models, Fields, Relationships

This is the start of a three-part tutorial. You'll build **`hcpi_outlet_onboarding`** — a small HCPI module the statistical office uses **back at the desk** to propose candidate outlets and log the visits made to them.

The real price-capture work happens on **tablets, in the field**, through the HCPI Flutter app — not in the web UI. The web is where supervisors do their back-office work, and outlet onboarding is a good example: someone hears about a new market, proposes it, a colleague visits it once or twice, a manager approves it, and only *then* does it appear in the collection rotation.

By the end of all three parts you'll have touched every basic concept in the HCPI back-end programme: models, the common field types and relationships, the ORM, the main view types, security, workflow, and chatter.

## What we're building, across all three parts

The story:

1. A supervisor hears about a new shop or market — call it a candidate outlet.
2. They create an **Outlet Proposal** with the name, address, GPS, contact, and outlet type (supermarket / open market / pharmacy …).
3. A field officer **visits** the location and logs what they saw.
4. A manager **approves** the proposal — the outlet becomes active and goes into the collection rotation. Later it may be **retired**.

| Part | Focus | What you end up with |
|---|---|---|
| **1 (this page)** | Module scaffolding, models, fields, relationships, ORM | A working module with two models and basic views, data you can create and query. |
| **[2: Views & UX](part2-views.md)** | All the main view types, search, list polish | Kanban, graph, pivot, calendar, filters, a coloured workflow form. |
| **[3: Security & Polish](part3-security.md)** | One custom role, sequence, chatter, validation | Production-ready: manager-only actions, audit trail, auto-numbered references. |

## The two models we're about to write

```
              ┌──────────────────────┐         ┌────────────────────┐
              │ hcpi.outlet.proposal │────────►│ hcpi.outlet.visit  │
              │     (the main one)   │   1:N   │   (child rows)     │
              └──────────────────────┘         └────────────────────┘
```

- **`hcpi.outlet.proposal`** — the main entity. One row per candidate outlet.
- **`hcpi.outlet.visit`** — child rows attached to a proposal (one or more field visits).

We'll also use Odoo's built-in `res.users` (the user model) on both sides — as a Many2one for "proposed by" / "visited by", and as a Many2many for "assigned team."

## What's in a module — a quick refresher

An Odoo module is just a folder under `addons_path` (configured in [`hcpi.conf`](../day1/configuration.md)) containing a few specific files. The skeleton:

```
hcpi_outlet_onboarding/
├── __init__.py            ← Python package marker; loads sub-packages
├── __manifest__.py        ← Odoo's metadata: name, dependencies, data files
├── models/                ← Python: classes that become database tables
├── security/              ← Who can do what
└── views/                 ← XML: how data is displayed in the browser
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

Delete the bits we don't need and add a file per model:

```bash
cd /opt/hcpi/custom/HCPI/hcpi_outlet_onboarding
rm -rf controllers demo
rm views/templates.xml models/models.py
touch models/hcpi_outlet_visit.py
touch models/hcpi_outlet_proposal.py
```

Final layout:

```
hcpi_outlet_onboarding/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   ├── hcpi_outlet_visit.py
│   └── hcpi_outlet_proposal.py
├── security/
│   └── ir.model.access.csv
└── views/
    └── views.xml
```

## Step 2: wire up `__init__.py` and the manifest

Open `models/__init__.py` and list both model files:

```python
from . import hcpi_outlet_visit
from . import hcpi_outlet_proposal
```

This file is plain Python. Each line tells Python "also import this sub-module when the package loads." If you forget a line, the corresponding model **silently won't load** — no error, just a missing model. This is the most common newcomer bug, so check this file first when something "doesn't exist."

The outer `__init__.py` (at the module root) already has `from . import models` from the scaffold — leave it.

Now `__manifest__.py`:

```python
{
    'name': 'HCPI Outlet Onboarding',
    'version': '18.0.1.0.0',
    'summary': 'Propose, visit, and approve candidate outlets.',
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
| `version` | Convention: `<odoo-major>.<your-major>.<minor>.<patch>`. |
| `summary` | One-line description on the Apps tile. |
| `category` | Group in the Apps screen. Free-form. |
| `depends` | Other modules required before this one can install. `base` is Odoo's foundation; `mail` gives us the chatter we'll wire up in Part 3. |
| `data` | XML/CSV files Odoo loads on install. **Order matters** — security first, then views. |
| `application` | `True` makes it appear as a top-level app in the launcher. |
| `license` | Required in modern Odoo. `LGPL-3` is the safe default. |

Notice **models aren't listed in `data`**. Models are Python — they load automatically through the `__init__.py` chain. Only XML/CSV files need to be enumerated.

## Step 3: the Visit model (the child)

Open `models/hcpi_outlet_visit.py`:

```python
from odoo import fields, models


class HcpiOutletVisit(models.Model):
    _name = 'hcpi.outlet.visit'
    _description = "Outlet Visit"
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
    visitor_id = fields.Many2one(
        'res.users',
        string="Visitor",
        default=lambda self: self.env.user,
        required=True,
    )
    result = fields.Selection(
        [
            ('open', "Open & Operating"),
            ('partial', "Partial / Issues"),
            ('closed', "Closed / Not Found"),
        ],
        default='open',
        required=True,
    )
    observations = fields.Text()
```

Walking through:

| Element | What it does |
|---|---|
| `_name = 'hcpi.outlet.visit'` | The technical name of the model. The PostgreSQL table will be `hcpi_outlet_visit` (dots become underscores). |
| `_description` | Human label. Required in modern Odoo. |
| `_order` | Default sort in lists. `date desc, id desc` puts the most recent visit first. |
| `Char` (name) | Short text — VARCHAR. `required=True` makes it mandatory. |
| `Many2one('hcpi.outlet.proposal', ...)` | A foreign key to another model. |
| `ondelete='cascade'` | "When the parent proposal is deleted, delete me too." Other options: `restrict` (block parent delete), `set null` (orphan me). |
| `Date` | Date without time. `fields.Date.context_today` is "today in the user's timezone." |
| `default=lambda self: self.env.user` | A function that runs at create-time and returns the current user. `self.env.user` is how you reach the current user from inside any model. |
| `Selection([...])` | A value picked from a fixed list. The tuples are `(database_value, label)`. Renders as a dropdown. |
| `Text` (observations) | Multi-line string — renders as a textarea. Use `Char` for short, `Text` for long. |

## Step 4: the Outlet Proposal model

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
    active = fields.Boolean(default=True)

    # Workflow
    state = fields.Selection(
        [
            ('draft', "Draft"),
            ('active', "Active"),
            ('retired', "Retired"),
        ],
        default='draft',
        required=True,
        tracking=True,
    )

    # Classification
    outlet_type = fields.Selection(
        [
            ('supermarket', "Supermarket"),
            ('open_market', "Open Market"),
            ('pharmacy', "Pharmacy"),
            ('convenience', "Convenience Store"),
            ('other', "Other"),
        ],
        required=True,
        default='open_market',
    )

    # Location / contact
    region = fields.Char(help="District or neighbourhood, e.g. Kampala / Central.")
    address = fields.Text()
    latitude = fields.Float(digits=(10, 6))
    longitude = fields.Float(digits=(10, 6))
    contact_name = fields.Char()
    contact_phone = fields.Char()

    # People
    proposed_by = fields.Many2one(
        'res.users',
        string="Proposed By",
        default=lambda self: self.env.user,
        required=True,
        index=True,
    )
    assigned_user_ids = fields.Many2many(
        'res.users',
        string="Assigned Team",
    )

    # Dates / notes
    opening_date = fields.Date(help="When the outlet enters the collection rotation.")
    notes = fields.Text()

    # Children
    visit_ids = fields.One2many(
        'hcpi.outlet.visit',
        'proposal_id',
        string="Visits",
    )

    # Computed
    visit_count = fields.Integer(
        compute='_compute_visit_count',
        store=True,
    )

    _sql_constraints = [
        (
            'name_region_unique',
            'unique(outlet_name, region)',
            "An outlet with this name in this region already exists.",
        ),
    ]

    @api.depends('visit_ids')
    def _compute_visit_count(self):
        for proposal in self:
            proposal.visit_count = len(proposal.visit_ids)
```

This is the bigger file — walk through the new patterns one section at a time.

| Pattern | What's new |
|---|---|
| `default="New"` + `readonly=True` + `copy=False` | `name` is a reference like `OP/2026/0001` — we'll generate it from a sequence in Part 3. For now `"New"` is a placeholder. `copy=False` means the value is *not* carried through when a user duplicates a record. |
| `active = fields.Boolean(default=True)` | `active` is a *magic* field name — Odoo treats records with `active=False` as archived and hides them from default searches. |
| `tracking=True` on `state` | Tells `mail.thread` to log every change in the chatter. We'll wire up `mail.thread` in Part 3 — `tracking=True` here is harmless until then. |
| `Selection([...])` (outlet_type) | Same as on the visit model — fixed list rendered as a dropdown. |
| `Float(digits=(10, 6))` | Decimal number. `digits=(precision, scale)` means up to 10 digits with 6 after the decimal point — perfect for GPS coordinates. |
| `Many2many('res.users', ...)` | Many-to-many via an auto-generated PostgreSQL join table. Here it points at the built-in user model — `assigned_user_ids` is the team monitoring this outlet. |
| `One2many(..., 'proposal_id', ...)` | The reverse side of the M2O on the visit model. `'proposal_id'` is the field on `hcpi.outlet.visit` that points back here. One2many has no DB column — it's computed from the other side. |
| `_sql_constraints` | Database-level constraints, enforced by PostgreSQL itself. Faster and more reliable than checking in Python because the database can't be bypassed. Format is `(name, sql_fragment, error_message)`. |
| `compute=...` + `store=True` + `@api.depends(...)` | A *stored computed field*. Odoo runs the method whenever any field listed in `@api.depends(...)` changes, and writes the result to the table. Because it's stored, you can search and group by it. |

### About the compute method

```python
@api.depends('visit_ids')
def _compute_visit_count(self):
    for proposal in self:
        proposal.visit_count = len(proposal.visit_ids)
```

**Always loop over `self`.** A compute is called on a *recordset* — could be one record, could be a thousand. You don't get to assume length-1.

## Step 5: open up security temporarily

Replace `security/ir.model.access.csv`:

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_hcpi_outlet_visit,hcpi.outlet.visit,model_hcpi_outlet_visit,base.group_user,1,1,1,1
access_hcpi_outlet_proposal,hcpi.outlet.proposal,model_hcpi_outlet_proposal,base.group_user,1,1,1,1
```

What's going on:

- One row per (model, group) pair.
- **`model_<model_table>`** is the auto-generated XML ID for each model's `ir.model` record. Pattern: take the model's table name (`hcpi.outlet.proposal` → `hcpi_outlet_proposal`) and prepend `model_`.
- **`base.group_user`** is the built-in group "every internal user."
- **`1,1,1,1`** = read, write, create, unlink (delete).

Every internal user gets full CRUD on both models. **We refine this in Part 3** — for now we want the lowest-friction setup so we can test the data layer.

## Step 6: minimal views to see the data

Real views come in Part 2 — for now we just need enough UI to create records and confirm the database is wired correctly. Open `views/views.xml` and replace its content:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <!-- ============ List ============ -->
    <record id="view_hcpi_outlet_proposal_list" model="ir.ui.view">
        <field name="name">hcpi.outlet.proposal.list</field>
        <field name="model">hcpi.outlet.proposal</field>
        <field name="arch" type="xml">
            <list>
                <field name="name"/>
                <field name="outlet_name"/>
                <field name="outlet_type"/>
                <field name="region"/>
                <field name="proposed_by"/>
                <field name="visit_count"/>
                <field name="state"/>
            </list>
        </field>
    </record>

    <!-- ============ Form ============ -->
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
                            <field name="outlet_type"/>
                            <field name="region"/>
                            <field name="proposed_by"/>
                            <field name="opening_date"/>
                        </group>
                        <group>
                            <field name="contact_name"/>
                            <field name="contact_phone"/>
                            <field name="latitude"/>
                            <field name="longitude"/>
                            <field name="visit_count"/>
                        </group>
                    </group>
                    <field name="assigned_user_ids" widget="many2many_tags"/>
                    <notebook>
                        <page string="Address">
                            <field name="address" placeholder="Street, landmark, neighbourhood..."/>
                        </page>
                        <page string="Visits">
                            <field name="visit_ids">
                                <list editable="bottom">
                                    <field name="date"/>
                                    <field name="name"/>
                                    <field name="visitor_id"/>
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

    <!-- ============ Action + Menu ============ -->
    <record id="action_hcpi_outlet_proposal" model="ir.actions.act_window">
        <field name="name">Outlet Proposals</field>
        <field name="res_model">hcpi.outlet.proposal</field>
        <field name="view_mode">list,form</field>
    </record>

    <menuitem id="menu_hcpi_outlet_onboarding_root"
              name="Outlet Onboarding"
              sequence="40"/>

    <menuitem id="menu_hcpi_outlet_proposals"
              name="Proposals"
              parent="menu_hcpi_outlet_onboarding_root"
              action="action_hcpi_outlet_proposal"
              sequence="10"/>

</odoo>
```

A few details worth flagging:

- **`<notebook>` and `<page>`** — tabs inside a form. Common pattern for "lots of related data on one form."
- **The inline `<list>` inside the One2many** — when rendering a One2many, Odoo needs to know which columns to show.
- **`<list editable="bottom">`** — lets you add visit rows directly in the list, like a spreadsheet. Saves opening a form for each one.
- **`widget="many2many_tags"`** on `assigned_user_ids` — renders the M2M as small chips with each user's name instead of a long dropdown.
- **Menus are records** — Odoo stores them in `ir.ui.menu`. `<menuitem>` is shorthand for creating that record.

## Step 7: install and click around

Start Odoo:

```bash
python odoo/odoo-bin -c conf/hcpi.conf
```

In the browser (developer mode on — see [User Administration](../day1/user-administration.md#activating-developer-mode)):

1. **Apps** → **Update Apps List** → confirm.
2. Search for **HCPI Outlet Onboarding** (remove the default "Apps" filter to see uninstalled modules).
3. Click **Activate**.
4. The **Outlet Onboarding** menu appears.

Open **Proposals → New**. Fill in:

- Outlet name: *Kikuubo Wholesale Market*
- Outlet type: *Open Market*
- Region: *Kampala / Central*
- Contact, GPS, opening date
- Assigned team: pick one or two users
- Switch to the **Visits** tab and add two visit rows with different results
- Save.

You should see the **Visits** count column on the list view (that's `visit_count`, the stored compute). Add another visit — the count updates after save.

## Step 8: meet the ORM via the Odoo shell

The ORM is the layer that lets you read and write database records without writing SQL. You use it constantly — in compute methods, button handlers, validations, everywhere.

Open a **second terminal** with the venv activated (leave Odoo running in the first), then:

```bash
cd /opt/hcpi
source venv/bin/activate
python odoo/odoo-bin shell -c conf/hcpi.conf
```

This drops you into a Python REPL with a fully loaded HCPI environment. The variable `env` is your gateway to everything.

```python
# Get a reference to a model (the "manager" — not a record)
Proposal = env['hcpi.outlet.proposal']

# Count all
Proposal.search_count([])

# Search by filter — the list of tuples is called a "domain"
markets = Proposal.search([('outlet_type', '=', 'open_market')])
markets.mapped('outlet_name')

# Cross-relation lookup (dot notation)
Proposal.search([('visit_ids.result', '=', 'closed')])

# Browse by id
p = Proposal.browse(1)
p.outlet_name
p.visit_ids.mapped('name')

# Create
new = Proposal.create({
    'outlet_name': 'Nakasero Pharmacy',
    'outlet_type': 'pharmacy',
    'region': 'Kampala / Central',
})

# Create with related children using a "command tuple"
Proposal.create({
    'outlet_name': 'Owino Market Stall 14',
    'outlet_type': 'open_market',
    'visit_ids': [
        (0, 0, {'name': 'First visit', 'result': 'partial'}),
        (0, 0, {'name': 'Follow-up', 'result': 'open'}),
    ],
})

# Update
new.write({'state': 'active'})

# Delete
new.unlink()

# Persist your shell changes
env.cr.commit()
```

What you've just used:

| Method | What it does |
|---|---|
| `search(domain)` | Find by filter — returns a recordset. |
| `search_count(domain)` | Count matching records. |
| `browse(ids)` | Fetch by known id(s). |
| `mapped('field')` | Pull one field from each record. |
| `create(values)` | Insert. |
| `write(values)` | Update — works on any recordset size. |
| `unlink()` | Delete. |

The `[(0, 0, {...})]` in `visit_ids` is Odoo's way of saying "create a new linked record." The full set:

| Tuple | Meaning |
|---|---|
| `(0, 0, values)` | Create a new linked record |
| `(1, id, values)` | Update existing linked record |
| `(2, id, 0)` | Unlink AND delete |
| `(3, id, 0)` | Unlink (don't delete) |
| `(4, id, 0)` | Link an existing record |
| `(5, 0, 0)` | Unlink all |
| `(6, 0, ids)` | Replace links with this list |

`(0, 0, ...)` (create) and `(6, 0, ids)` (replace) are the two most common.

## Quick check

Try to answer without scrolling up.

**1.** Inside the `models/` folder, which file tells Python which sub-modules to load?

??? success "Answer"
    `__init__.py`. One `from . import <filename>` line per model file. Forget a line and the model silently won't load.

**2.** The model `_name = 'hcpi.outlet.visit'` produces which PostgreSQL table name?

??? success "Answer"
    `hcpi_outlet_visit`. Dots in the `_name` become underscores in the table.

**3.** Which field type would you use for a long, multi-line address?

??? success "Answer"
    `fields.Text`. Use `fields.Char` for short single-line text, `fields.Text` for multi-line.

**4.** Which field type would you use for GPS latitude — a decimal number?

??? success "Answer"
    `fields.Float`, typically with `digits=(10, 6)` so you get six decimal places.

**5.** What does `ondelete='cascade'` on a Many2one mean?

??? success "Answer"
    When the record on the *other* side (the parent) is deleted, this record is deleted too. Other options: `restrict` (block the parent delete) and `set null` (orphan this record).

**6.** In `One2many('hcpi.outlet.visit', 'proposal_id', ...)`, what is the `'proposal_id'` string?

??? success "Answer"
    The name of the Many2one field on the *other* model (`hcpi.outlet.visit`) that points back here. One2many has no database column — it's calculated by walking the inverse Many2one.

**7.** In a `@api.depends` compute method, why do you always write `for record in self`?

??? success "Answer"
    Because `self` is a *recordset* of any size — could be one record, could be a thousand. You can't assume length-1.

**8.** Which ORM method deletes records — `delete()`, `remove()`, or `unlink()`?

??? success "Answer"
    `unlink()`. (Historical reason: it mirrors the Unix `unlink()` system call.)

## Exercises

1. **Add a `weekday` computed field** to `hcpi.outlet.visit` — a `Char` showing `"Monday"`, `"Tuesday"`, etc. derived from `date`. Hint: `fields.Date` values are Python `date` objects; they have `.strftime('%A')`. Use `@api.depends('date')` and skip `store=True`.

2. **Bulk update.** In the shell, set `state = 'active'` for every draft proposal that already has at least one visit. One ORM call.

3. **Find proposals with no team assigned.** Write a domain in the shell. Hint: the operator is `'='` against `False` for empty Many2many.

4. **Add a `priority` field**: `Selection([('0', 'Normal'), ('1', 'High'), ('2', 'Urgent')])` on the proposal model. Default to `'0'`. We'll wire it into the form next part.

## What you learned

After Part 1 you've covered the entire data layer of an HCPI module:

- **Module scaffolding**: manifest, `__init__.py` package chain, `data` files, `depends`.
- **Field types**: `Char`, `Text`, `Integer`, `Float` (with `digits=`), `Boolean`, `Date`, `Selection`.
- **Relationships**: `Many2one` (with `ondelete=`), `One2many` (with the inverse field name), `Many2many` (to a built-in model).
- **Computed fields**: stored, `@api.depends`, the for-loop-over-self pattern.
- **`_sql_constraints`** for database-level uniqueness.
- **Minimal views**: list, form, notebook + page, inline One2many, menus.
- **CSV-format access control** — open today, locked down in Part 3.
- **The ORM**: `search`, `browse`, `create`, `write`, `unlink`, `mapped`, and command tuples.

## What's next

➡️ **[Part 2: Views & UX](part2-views.md)** — turn this raw module into something pleasant to use. Statusbar workflow, Kanban with stages, Graph and Pivot reports, Calendar, advanced search filters, and list decorations.
