# Building an HCPI Module — Part 2: Views & UX

<!--
Duration: one day (about 7 hours including breaks).
Covers: form view structure (statusbar, sheet, notebook, smart buttons),
Kanban with stages, Graph, Pivot, Calendar, Search views with filters and
group bys, list view decorations and actions, QWeb PDF reports.
A short look at OWL (Odoo's JS framework) at the end.
Prereq: Part 1 complete — hcpi_outlet_onboarding installed with data.
-->

You've got a module with data flowing in and out. Now we make it pleasant to use.

## A quick tour of every view type

| View | XML tag | When to use it |
|---|---|---|
| **List** (a.k.a. Tree) | `<list>` | Tabular — many records at once. The default. |
| **Form** | `<form>` | One record at a time, fields laid out for editing. |
| **Search** | `<search>` | The filter sidebar with predefined filters + group bys. |
| **Kanban** | `<kanban>` | Cards grouped by a field (typically `state`). Workflow boards. |
| **Graph** | `<graph>` | Bar / line / pie charts over aggregated data. |
| **Pivot** | `<pivot>` | Spreadsheet-style cross-tabs. |
| **Calendar** | `<calendar>` | Records placed on a date. Day/week/month switcher. |
| **Activity** | `<activity>` | Records with their pending CRM-style activities. |
| **Gantt** | `<gantt>` | Records with start/end dates as bars. |

We'll build the first seven and just mention the rest. That's already more than most HCPI modules need.

## Step 1: split the views into one file per model

The single `views/views.xml` from Part 1 worked but won't scale. Re-organise:

```bash
cd /opt/hcpi/custom/HCPI/hcpi_outlet_onboarding/views
mv views.xml hcpi_outlet_proposal_views.xml
touch hcpi_region_views.xml
touch hcpi_outlet_type_views.xml
touch hcpi_outlet_tag_views.xml
touch hcpi_menus.xml
```

Then move the menus and the region / type / tag views into their own files. Update `__manifest__.py` `data` list to enumerate them:

```python
'data': [
    'security/ir.model.access.csv',
    'views/hcpi_region_views.xml',
    'views/hcpi_outlet_type_views.xml',
    'views/hcpi_outlet_tag_views.xml',
    'views/hcpi_outlet_proposal_views.xml',
    'views/hcpi_menus.xml',
],
```

Order matters: views must be loaded **before** the menus that reference their actions.

!!! tip "One file per model is the HCPI convention"
    Look at `hcpi_outlet` — it has `hcpi_outlet_views.xml`, `hcpi_outlet_menus.xml`, plus separate files for sub-models. Match the convention so other devs can find things.

## Step 2: a proper form view with a statusbar and smart buttons

Open `views/hcpi_outlet_proposal_views.xml` and replace the form view with:

```xml
<record id="view_hcpi_outlet_proposal_form" model="ir.ui.view">
    <field name="name">hcpi.outlet.proposal.form</field>
    <field name="model">hcpi.outlet.proposal</field>
    <field name="arch" type="xml">
        <form>
            <header>
                <button name="action_start_inspection"
                        string="Start Inspection"
                        type="object"
                        class="btn-primary"
                        invisible="state != 'draft'"/>
                <button name="action_submit_for_review"
                        string="Submit for Review"
                        type="object"
                        class="btn-primary"
                        invisible="state != 'inspecting'"/>
                <button name="action_approve"
                        string="Approve"
                        type="object"
                        class="btn-primary"
                        invisible="state != 'review'"/>
                <button name="action_reject"
                        string="Reject"
                        type="object"
                        invisible="state not in ('review', 'inspecting')"/>
                <button name="action_reset_to_draft"
                        string="Reset to Draft"
                        type="object"
                        invisible="state == 'draft'"/>
                <field name="state" widget="statusbar"
                       statusbar_visible="draft,inspecting,review,approved"/>
            </header>
            <sheet>
                <div class="oe_button_box" name="button_box">
                    <button name="action_view_inspections"
                            type="object"
                            class="oe_stat_button"
                            icon="fa-search">
                        <field name="inspection_count" widget="statinfo" string="Inspections"/>
                    </button>
                </div>
                <div class="oe_title">
                    <label for="outlet_name"/>
                    <h1><field name="outlet_name" placeholder="e.g. Kikuubo Wholesale Market"/></h1>
                    <span class="text-muted"><field name="name" readonly="1"/></span>
                </div>
                <group>
                    <group string="Classification">
                        <field name="region_id"/>
                        <field name="outlet_type_id"/>
                        <field name="proposed_by"/>
                        <field name="reviewed_by" readonly="1"/>
                        <field name="approval_date" readonly="1"/>
                    </group>
                    <group string="Location &amp; Contact">
                        <field name="contact_name"/>
                        <field name="contact_phone"/>
                        <field name="operating_hours"/>
                        <label for="latitude" string="GPS"/>
                        <div>
                            <field name="latitude" class="oe_inline"/>,
                            <field name="longitude" class="oe_inline"/>
                        </div>
                        <field name="has_failed_inspection" readonly="1"/>
                    </group>
                </group>
                <field name="tag_ids" widget="many2many_tags" options="{'color_field': 'color'}"/>
                <notebook>
                    <page string="Address" name="address">
                        <field name="address" placeholder="Street, landmark, neighbourhood..."/>
                    </page>
                    <page string="Inspections" name="inspections">
                        <field name="inspection_ids">
                            <list editable="bottom">
                                <field name="date"/>
                                <field name="name"/>
                                <field name="inspector_id"/>
                                <field name="result" widget="badge"
                                       decoration-danger="result == 'fail'"
                                       decoration-warning="result == 'partial'"
                                       decoration-success="result == 'pass'"/>
                            </list>
                            <form>
                                <sheet>
                                    <group>
                                        <field name="name"/>
                                        <field name="date"/>
                                        <field name="inspector_id"/>
                                        <field name="result"/>
                                    </group>
                                    <field name="photo" widget="image" class="oe_avatar"/>
                                    <field name="observations" placeholder="What did you see?"/>
                                </sheet>
                            </form>
                        </field>
                    </page>
                    <page string="Notes" name="notes">
                        <field name="notes"/>
                    </page>
                </notebook>
            </sheet>
        </form>
    </field>
</record>
```

A lot is new here — walk through it section by section.

### `<header>` and the statusbar

```xml
<header>
    <button name="action_start_inspection" ... invisible="state != 'draft'"/>
    ...
    <field name="state" widget="statusbar" statusbar_visible="draft,inspecting,review,approved"/>
</header>
```

The header sits above the sheet. Conventions:

- **Workflow buttons** on the left.
- **`<field name="state" widget="statusbar">`** on the right — renders the breadcrumb-style progress display.
- **`invisible="<expression>"`** — Odoo 17/18 syntax. The expression is a Python-ish condition evaluated client-side. Old Odoo used `attrs="{'invisible': [(...)]}"`; you'll still see both in HCPI code.
- **`statusbar_visible="draft,inspecting,review,approved"`** — show only those four states in the bar (hide "rejected" unless current). Optional polish.

The buttons call Python methods via `name="action_start_inspection" type="object"`. We need to add those methods now. Open `models/hcpi_outlet_proposal.py` and add at the bottom of the class:

```python
def action_start_inspection(self):
    for proposal in self:
        proposal.state = 'inspecting'

def action_submit_for_review(self):
    for proposal in self:
        proposal.state = 'review'

def action_approve(self):
    for proposal in self:
        proposal.state = 'approved'
        proposal.reviewed_by = self.env.user
        proposal.approval_date = fields.Date.context_today(proposal)

def action_reject(self):
    for proposal in self:
        proposal.state = 'rejected'
        proposal.reviewed_by = self.env.user

def action_reset_to_draft(self):
    for proposal in self:
        proposal.state = 'draft'
        proposal.reviewed_by = False
        proposal.approval_date = False

def action_view_inspections(self):
    self.ensure_one()
    return {
        'type': 'ir.actions.act_window',
        'name': "Inspections",
        'res_model': 'hcpi.outlet.inspection',
        'view_mode': 'list,form',
        'domain': [('proposal_id', '=', self.id)],
        'context': {'default_proposal_id': self.id},
    }
```

Things to notice:

- **`for proposal in self`** — always loop, even if a button is usually clicked on one record at a time. Lists support multi-selection.
- **`self.ensure_one()`** — used in `action_view_inspections` because it only makes sense on a single record (it builds a window-action). Raises if you have more than one.
- **The dict returned from `action_view_inspections`** — that's an Odoo "client action" descriptor. Returning a dict like this from a button tells the UI to navigate. The `default_proposal_id` in `context` pre-fills that field when you create a new inspection from the resulting view.
- **`reviewed_by = self.env.user`** — set automatically on approval/rejection so you have an audit trail for free.

Hardened validation (you can't submit empty, only supervisors can approve, etc.) comes in Part 3 — keep it simple for now.

### Smart buttons (the box at the top of the form)

```xml
<div class="oe_button_box" name="button_box">
    <button name="action_view_inspections"
            type="object"
            class="oe_stat_button"
            icon="fa-search">
        <field name="inspection_count" widget="statinfo" string="Inspections"/>
    </button>
</div>
```

**Smart buttons** are those large counters you see on a partner form ("12 Sales Orders", "3 Invoices"). The pattern is:

- `oe_button_box` div in the top-right of the sheet.
- Buttons with class `oe_stat_button` and an icon (Font Awesome — anything starting with `fa-`).
- Inside each button, a `<field widget="statinfo">` showing the count.
- Clicking opens a list of related records.

We already show inspections inline in the notebook — the smart button is an alternative way to focus on just them in their own view, useful when you have many.

### List view inside the One2many — with decorations

```xml
<field name="inspection_ids">
    <list editable="bottom">
        ...
        <field name="result" widget="badge"
               decoration-danger="result == 'fail'"
               decoration-warning="result == 'partial'"
               decoration-success="result == 'pass'"/>
    </list>
    <form>
        ...
    </form>
</field>
```

Two things:

1. **`widget="badge"`** + `decoration-*` — renders the field as a coloured pill. The decoration classes (`danger`, `warning`, `info`, `success`, `muted`) map to Bootstrap colours.
2. **An inline `<form>`** alongside the `<list>` — when the user double-clicks an inspection row, this form opens in a modal. Without an inline form, double-click would open a default tiny one.

You can also use `decoration-*` on `<list>` rows themselves (`<list decoration-danger="result == 'fail'">`) to highlight whole rows.

## Step 3: install / upgrade and check the form

```bash
python odoo/odoo-bin -c conf/hcpi.conf -u hcpi_outlet_onboarding
```

Open a proposal. The form should now have:

- A status bar at the top with **Draft → Inspecting → Under Review → Approved**.
- Workflow buttons that change based on the current state.
- A smart button counting inspections.
- A coloured result badge in the inspection list.

Click **Start Inspection** → the state advances and the next button appears. Click **Submit for Review** → **Approve** and **Reject** appear. Click **Approve** → state moves to "Approved", and `reviewed_by` / `approval_date` populate from the Python method.

## Step 4: Kanban view

Kanban is the workflow-board view. Cards grouped by `state` (or any field), drag-to-update.

Add this view to `hcpi_outlet_proposal_views.xml` *and* extend the action to include `kanban`:

```xml
<record id="view_hcpi_outlet_proposal_kanban" model="ir.ui.view">
    <field name="name">hcpi.outlet.proposal.kanban</field>
    <field name="model">hcpi.outlet.proposal</field>
    <field name="arch" type="xml">
        <kanban default_group_by="state" class="o_kanban_small_column">
            <field name="name"/>
            <field name="outlet_name"/>
            <field name="region_id"/>
            <field name="outlet_type_id"/>
            <field name="proposed_by"/>
            <field name="inspection_count"/>
            <field name="has_failed_inspection"/>
            <field name="tag_ids"/>
            <templates>
                <t t-name="card">
                    <div class="oe_kanban_global_click">
                        <div class="o_kanban_record_title">
                            <strong><field name="outlet_name"/></strong>
                        </div>
                        <div class="o_kanban_record_subtitle">
                            <field name="region_id"/> · <field name="outlet_type_id"/>
                        </div>
                        <div class="mt-2">
                            <field name="tag_ids" widget="many2many_tags" options="{'color_field': 'color'}"/>
                        </div>
                        <div class="o_kanban_record_bottom mt-2">
                            <div class="oe_kanban_bottom_left">
                                <i class="fa fa-user me-1"/>
                                <field name="proposed_by"/>
                            </div>
                            <div class="oe_kanban_bottom_right">
                                <span t-if="record.has_failed_inspection.raw_value" class="badge text-bg-danger me-1">
                                    Failed inspection
                                </span>
                                <span class="badge text-bg-info">
                                    <i class="fa fa-search me-1"/>
                                    <field name="inspection_count"/>
                                </span>
                            </div>
                        </div>
                    </div>
                </t>
            </templates>
        </kanban>
    </field>
</record>
```

And update the action:

```xml
<record id="action_hcpi_outlet_proposal" model="ir.actions.act_window">
    <field name="name">Outlet Proposals</field>
    <field name="res_model">hcpi.outlet.proposal</field>
    <field name="view_mode">kanban,list,form,graph,pivot,calendar</field>
</record>
```

The action's `view_mode` lists all the view types this action can switch to. The order also sets the **default** view (kanban first → kanban opens by default).

What's going on inside the kanban:

- **`default_group_by="state"`** — start grouped by state. The columns are the values of `state`. Drag a card across columns to write that value.
- **`<field>` declarations at the top** — every field used inside the template must be declared. The kanban renderer only fetches declared fields.
- **`<templates><t t-name="card">`** — Odoo 18's modern kanban uses the `card` template name. (Older code uses `kanban-box`.) Inside the `<t>` you write standard QWeb-flavoured HTML.
- **`record.has_failed_inspection.raw_value`** — inside kanban, `record.<fieldname>` gives you `.value` (formatted) and `.raw_value` (the actual stored value). For booleans you want `raw_value` for truthiness checks.

Upgrade and open **Outlet Onboarding → Proposals**. You should land on a kanban board with columns for each state. Drag a card from Draft to Inspecting — the state writes.

!!! tip "If kanban doesn't show columns"
    Make sure your `state` field is a `Selection` (it is). Make sure you have records — empty kanban shows the action's "help" placeholder, not the columns.

## Step 5: Graph view

```xml
<record id="view_hcpi_outlet_proposal_graph" model="ir.ui.view">
    <field name="name">hcpi.outlet.proposal.graph</field>
    <field name="model">hcpi.outlet.proposal</field>
    <field name="arch" type="xml">
        <graph string="Outlet Proposals" type="bar" sample="1">
            <field name="region_id"/>
            <field name="outlet_type_id"/>
            <field name="inspection_count" type="measure"/>
        </graph>
    </field>
</record>
```

- **`type="bar"`** — also `line`, `pie`.
- **`<field type="measure">`** — what to aggregate. Default aggregation is sum.
- A field without `type="measure"` is a **group-by axis**.
- **`sample="1"`** — show generated sample data when the real table is empty (handy during demos).

Click the **Graph** view selector in the Proposals list → see a bar chart of total inspections per region, broken down by outlet type.

## Step 6: Pivot view

```xml
<record id="view_hcpi_outlet_proposal_pivot" model="ir.ui.view">
    <field name="name">hcpi.outlet.proposal.pivot</field>
    <field name="model">hcpi.outlet.proposal</field>
    <field name="arch" type="xml">
        <pivot string="Outlet Proposals">
            <field name="region_id" type="row"/>
            <field name="state" type="col"/>
            <field name="inspection_count" type="measure"/>
        </pivot>
    </field>
</record>
```

Same idea but two-dimensional. Rows = regions, columns = states, cells = sum of inspections.

The user can click `+` on a row to expand it (by outlet type, by inspector, etc.) and the pivot recomputes. **Without writing any code** they can pivot, filter, and group by anything.

## Step 7: Calendar view

```xml
<record id="view_hcpi_outlet_proposal_calendar" model="ir.ui.view">
    <field name="name">hcpi.outlet.proposal.calendar</field>
    <field name="model">hcpi.outlet.proposal</field>
    <field name="arch" type="xml">
        <calendar string="Outlet Proposals"
                  date_start="approval_date"
                  color="region_id"
                  mode="month">
            <field name="outlet_name"/>
            <field name="region_id"/>
        </calendar>
    </field>
</record>
```

- **`date_start="approval_date"`** — the field that places the record on the calendar. (Approved proposals will appear on their approval day.)
- **`color="region_id"`** — colour each entry by region. With a Many2one, Odoo assigns colours automatically.
- **`mode="month"`** — `day` / `week` / `month` / `year`. The user can switch.

A nicer alternative would be a calendar over the *inspections* showing each visit on the day it happened — a good exercise.

## Step 8: Search view — filters and group bys

```xml
<record id="view_hcpi_outlet_proposal_search" model="ir.ui.view">
    <field name="name">hcpi.outlet.proposal.search</field>
    <field name="model">hcpi.outlet.proposal</field>
    <field name="arch" type="xml">
        <search>
            <!-- Quick text search across these fields -->
            <field name="outlet_name"/>
            <field name="name"/>
            <field name="proposed_by"/>
            <field name="region_id"/>
            <field name="outlet_type_id"/>
            <field name="tag_ids"/>

            <!-- Predefined filters (one-click) -->
            <filter name="draft" string="Draft" domain="[('state', '=', 'draft')]"/>
            <filter name="inspecting" string="Inspecting" domain="[('state', '=', 'inspecting')]"/>
            <filter name="review" string="Under Review" domain="[('state', '=', 'review')]"/>
            <filter name="approved" string="Approved" domain="[('state', '=', 'approved')]"/>
            <separator/>
            <filter name="mine" string="My Proposals"
                    domain="[('proposed_by', '=', uid)]"/>
            <filter name="this_month" string="Submitted This Month"
                    domain="[('create_date', '&gt;=', (context_today() + relativedelta(day=1)).strftime('%Y-%m-%d'))]"/>
            <filter name="failed_inspection" string="Has Failed Inspection"
                    domain="[('has_failed_inspection', '=', True)]"/>
            <filter name="no_inspections" string="No Inspections Yet"
                    domain="[('inspection_count', '=', 0)]"/>

            <!-- Group-by options -->
            <group expand="0" string="Group By">
                <filter name="group_state" string="State"
                        context="{'group_by': 'state'}"/>
                <filter name="group_region" string="Region"
                        context="{'group_by': 'region_id'}"/>
                <filter name="group_type" string="Outlet Type"
                        context="{'group_by': 'outlet_type_id'}"/>
                <filter name="group_proposed_by" string="Proposed By"
                        context="{'group_by': 'proposed_by'}"/>
            </group>
        </search>
    </field>
</record>
```

Important pieces:

- **`<field>` at the top** — fields searchable from the quick-search box. Typing in the search bar searches all of them.
- **`<filter>`** — a named toggle. Click to apply the `domain`. Multiple active filters on the same field OR together; across fields AND together.
- **`<separator/>`** — a visual divider in the filter dropdown.
- **`<group expand="0">`** — wraps the group-by options. `expand="0"` keeps the section collapsed by default.
- **`context="{'group_by': 'state'}"`** — what makes a filter act as a group-by instead of a domain.
- **`uid`** — a magic variable referring to the current user's id. Use it for "my X" filters.
- **`context_today()` and `relativedelta(...)`** — date helpers Odoo exposes inside domain expressions, for things like "this month."

Once you reload, the Proposals list has the new search sidebar. Try:

- Click **My Proposals** + **Submitted This Month** → narrowed to your own recent work.
- Group By **Region** → list collapses into region groups with counts.
- Group By **Region** then **Outlet Type** (nested) → two-level grouping.

## Step 9: list view decorations and a server action

Update the list view inside `hcpi_outlet_proposal_views.xml`:

```xml
<record id="view_hcpi_outlet_proposal_list" model="ir.ui.view">
    <field name="name">hcpi.outlet.proposal.list</field>
    <field name="model">hcpi.outlet.proposal</field>
    <field name="arch" type="xml">
        <list decoration-info="state == 'inspecting'"
              decoration-warning="state == 'review'"
              decoration-success="state == 'approved'"
              decoration-muted="state == 'draft'"
              decoration-danger="state == 'rejected'">
            <field name="name"/>
            <field name="outlet_name"/>
            <field name="region_id" optional="show"/>
            <field name="outlet_type_id" optional="show"/>
            <field name="proposed_by" optional="show"/>
            <field name="inspection_count" optional="show" sum="Total"/>
            <field name="last_inspection_date" optional="hide"/>
            <field name="has_failed_inspection" optional="hide" widget="boolean_toggle"/>
            <field name="state" widget="badge"
                   decoration-info="state == 'inspecting'"
                   decoration-warning="state == 'review'"
                   decoration-success="state == 'approved'"
                   decoration-muted="state == 'draft'"
                   decoration-danger="state == 'rejected'"/>
        </list>
    </field>
</record>
```

- **Row-level decorations** colour the whole row by state.
- **`sum="Total"`** on numeric columns shows a sum row at the bottom (or below the current group when grouped).
- **`optional="show"` / `optional="hide"`** — column can be toggled by the user from the column menu.
- **`widget="boolean_toggle"`** — boolean shown as a slider rather than a checkbox.

### A server action accessible from the list

Add this near the top of `hcpi_outlet_proposal_views.xml`:

```xml
<record id="action_server_submit_selected" model="ir.actions.server">
    <field name="name">Submit Selected for Review</field>
    <field name="model_id" ref="model_hcpi_outlet_proposal"/>
    <field name="binding_model_id" ref="model_hcpi_outlet_proposal"/>
    <field name="binding_view_types">list</field>
    <field name="state">code</field>
    <field name="code">
records.filtered(lambda r: r.state == 'inspecting').write({'state': 'review'})
    </field>
</record>
```

- **`ir.actions.server` + `state='code'`** — server action that runs the Python in `code`.
- **`binding_model_id`** + **`binding_view_types="list"`** — adds this action to the **Actions** drop-down on the list view, only visible there.
- **`records`** — magic variable in server-action code: the selected recordset.
- **`.filtered(lambda r: ...)`** — a recordset method that returns only matching records. Same idea as Python's `filter()`.

After upgrade, select multiple proposals in the list → **Actions ▾ → Submit Selected for Review** appears. (Click multiple checkboxes first.)

## Step 10: a printable PDF dossier

Reports in Odoo are **QWeb** templates rendered to PDF. Create a folder and template:

```bash
mkdir -p /opt/hcpi/custom/HCPI/hcpi_outlet_onboarding/reports
touch /opt/hcpi/custom/HCPI/hcpi_outlet_onboarding/reports/hcpi_outlet_proposal_report.xml
```

Add the path to `__manifest__.py` `data`:

```python
'data': [
    'security/ir.model.access.csv',
    'reports/hcpi_outlet_proposal_report.xml',
    'views/hcpi_region_views.xml',
    ...
],
```

Then write the report template (`reports/hcpi_outlet_proposal_report.xml`):

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <!-- The report action — what makes it appear in the Print menu -->
    <record id="action_report_hcpi_outlet_proposal" model="ir.actions.report">
        <field name="name">Outlet Onboarding Dossier</field>
        <field name="model">hcpi.outlet.proposal</field>
        <field name="report_type">qweb-pdf</field>
        <field name="report_name">hcpi_outlet_onboarding.report_outlet_proposal_document</field>
        <field name="report_file">hcpi_outlet_onboarding.report_outlet_proposal_document</field>
        <field name="binding_model_id" ref="model_hcpi_outlet_proposal"/>
        <field name="binding_type">report</field>
    </record>

    <!-- The QWeb template itself -->
    <template id="report_outlet_proposal_document">
        <t t-call="web.html_container">
            <t t-foreach="docs" t-as="proposal">
                <t t-call="web.external_layout">
                    <div class="page">
                        <h2>
                            Outlet Onboarding Dossier —
                            <span t-field="proposal.outlet_name"/>
                        </h2>
                        <div class="row mb-3">
                            <div class="col-6">
                                <strong>Reference:</strong> <span t-field="proposal.name"/><br/>
                                <strong>Region:</strong> <span t-field="proposal.region_id.complete_name"/><br/>
                                <strong>Type:</strong> <span t-field="proposal.outlet_type_id"/><br/>
                                <strong>Operating hours:</strong> <span t-field="proposal.operating_hours"/>
                            </div>
                            <div class="col-6">
                                <strong>Proposed by:</strong> <span t-field="proposal.proposed_by"/><br/>
                                <strong>State:</strong> <span t-field="proposal.state"/><br/>
                                <strong>Contact:</strong> <span t-field="proposal.contact_name"/> — <span t-field="proposal.contact_phone"/><br/>
                                <strong>GPS:</strong> <span t-field="proposal.latitude"/>, <span t-field="proposal.longitude"/>
                            </div>
                        </div>

                        <h4>Address</h4>
                        <p t-field="proposal.address"/>

                        <h4>Inspections</h4>
                        <table class="table table-sm">
                            <thead>
                                <tr>
                                    <th>Date</th>
                                    <th>Summary</th>
                                    <th>Inspector</th>
                                    <th>Result</th>
                                </tr>
                            </thead>
                            <tbody>
                                <tr t-foreach="proposal.inspection_ids" t-as="insp">
                                    <td><span t-field="insp.date"/></td>
                                    <td><span t-field="insp.name"/></td>
                                    <td><span t-field="insp.inspector_id"/></td>
                                    <td>
                                        <span t-attf-class="badge text-bg-#{ {'pass': 'success', 'partial': 'warning', 'fail': 'danger'}.get(insp.result, 'secondary') }">
                                            <span t-field="insp.result"/>
                                        </span>
                                    </td>
                                </tr>
                            </tbody>
                        </table>

                        <t t-if="proposal.notes">
                            <h4>Notes</h4>
                            <p t-field="proposal.notes"/>
                        </t>
                    </div>
                </t>
            </t>
        </t>
    </template>

</odoo>
```

The pieces:

| Element | Purpose |
|---|---|
| `ir.actions.report` | Registers the report. Adds it to the **Print** menu on the form. |
| `binding_model_id` + `binding_type="report"` | Wires it under "Print" specifically for `hcpi.outlet.proposal`. |
| `web.html_container` | Standard wrapper providing CSS resets and page sizing. |
| `web.external_layout` | Adds header (company name/logo) and footer. Use `web.internal_layout` to skip. |
| `t-foreach`, `t-as` | Loop over records (`docs` is the recordset passed in). |
| `t-field="proposal.outlet_name"` | Renders the field value with formatting (dates, monetary, etc.). |
| `t-attf-class="...#{ expr }..."` | String formatting inside an attribute (the `#{}` is QWeb syntax). |
| `t-if` / `t-else` | Conditionals. |

After upgrade, open a proposal → click **Print ▾ → Outlet Onboarding Dossier**. A PDF downloads with the formatted dossier.

!!! tip "QWeb is HTML-with-superpowers"
    QWeb templates are evaluated server-side and produce HTML. Browser-side, you can use the *same* QWeb in OWL components and kanban templates. Same syntax, different runtime.

## Step 11: a brief look at OWL (Odoo's frontend framework)

You won't write OWL components in this tutorial — that's a deeper rabbit hole — but you should know what it is and where to look.

**OWL** (Odoo Web Library) is Odoo's reactive component framework, conceptually similar to Vue or React but with Odoo-tuned ergonomics. Every form view, kanban card, statusbar, etc. that you've used in this tutorial is rendered by OWL components.

You meet OWL when you:

- **Write a custom widget** — e.g., a field that displays an interactive map for `latitude`/`longitude`. Add it via `field_registry.add('my_widget', MyWidget)` in a JS file under `static/src/`.
- **Override an existing component** — change how the kanban card renders for a specific model, e.g., to add a chart inline.
- **Build a custom view** — `ir.actions.client` views (think dashboards) are pure OWL.

What it looks like (you don't need to type this — just recognise it):

```javascript
/** @odoo-module */
import { Component } from "@odoo/owl";
import { registry } from "@web/core/registry";

class FieldResultBadge extends Component {
    static template = "hcpi_outlet_onboarding.ResultBadge";
    get colorClass() {
        return {
            pass: "text-bg-success",
            partial: "text-bg-warning",
            fail: "text-bg-danger",
        }[this.props.record.data.result] || "text-bg-secondary";
    }
}

registry.category("fields").add("result_badge", { component: FieldResultBadge });
```

The takeaway: **most Odoo development never touches OWL.** You build models, write views in XML, and the framework gives you a rich UI for free. OWL is the escape hatch for the 5% case — a custom map widget for those `latitude`/`longitude` fields would be a perfect example.

## Exercises

1. **Inspection-based calendar.** Add a Calendar view to `hcpi.outlet.inspection` with `date_start="date"`. Expose it via a new action and menu. (Each visit then shows on the day it happened, colour-coded by inspector.)

2. **Custom group-by**: add a group-by filter for "Has Failed Inspection" — toggle the search to group proposals into Yes/No.

3. **Server action that uses `mapped`**: write an `ir.actions.server` that on selected proposals, raises a friendly summary via `UserError` like `"Selected: 5 proposals, 18 inspections, 2 with failed inspections."`. Hint: `sum(records.mapped('inspection_count'))` and `len(records.filtered(...))`.

4. **Override the report layout**: include the photo from the first inspection (if any) inside the dossier. Hint: `proposal.inspection_ids[:1]` gives the first inspection as a recordset; `<img t-att-src="image_data_uri(insp.photo)"/>` displays it.

5. **Add a `priority` field**: `Selection([('0', 'Normal'), ('1', 'High'), ('2', 'Urgent')])` with `widget="priority"` on the form (it renders as stars). Add it to the kanban card top-right corner.

## What you learned

After Part 2 you understand:

- **Every view type** (List, Form, Search, Kanban, Graph, Pivot, Calendar) and when to reach for each.
- **Form composition**: header + statusbar + smart buttons + sheet + notebook + pages.
- **Workflow buttons** that call Python methods via `name="..."` + `type="object"`.
- **Inline One2many editing** in lists, plus an inline form for the modal editor.
- **Search views**: domain filters, "my X" with `uid`, date helpers, separators, group-by.
- **List decorations**, sums, optional columns, badges, boolean toggles.
- **Server actions** with `binding_*` to expose them in the UI.
- **QWeb PDF reports**: `ir.actions.report`, `web.external_layout`, `t-foreach`, `t-field`, `t-if`, `t-attf-class`.
- **What OWL is** and when you'd need to learn it deeper.

## What's next

➡️ **[Part 3: Security & Polish](part3-security.md)** — close the module. Define field-officer, supervisor, and manager groups; restrict access at the model and record level; add a sequence for auto-naming; wire `mail.thread` for chatter and audit trail; and ship a production-ready module.
