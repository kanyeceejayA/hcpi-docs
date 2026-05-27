# Building HCPI Field Reports — Part 2: Views & UX

**Duration:** one day (about 7 hours including breaks).
**Covers:** form view structure (statusbar, sheet, notebook, smart buttons), Kanban with stages, Graph, Pivot, Calendar, Search views with filters and group bys, list view decorations and actions, QWeb PDF reports. A short look at OWL (Odoo's JS framework) at the end.

You've got a module with data flowing in and out. Now we make it pleasant to use.

!!! info "Before you start"
    Part 1 of this tutorial is complete: you have `hcpi_field_reports` installed and a handful of records in the database. If not, do [Part 1](part1-models.md) first.

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
cd /opt/hcpi/custom/HCPI/hcpi_field_reports/views
mv views.xml hcpi_field_report_views.xml
touch hcpi_region_views.xml hcpi_field_tag_views.xml hcpi_menus.xml
```

Then move the menus and region/tag views into their own files. Update `__manifest__.py` `data` list:

```python
'data': [
    'security/ir.model.access.csv',
    'views/hcpi_region_views.xml',
    'views/hcpi_field_tag_views.xml',
    'views/hcpi_field_report_views.xml',
    'views/hcpi_menus.xml',
],
```

Order matters: views must be loaded **before** the menus that reference their actions.

!!! tip "One file per model is the HCPI convention"
    Look at `hcpi_outlet` — it has `hcpi_outlet_views.xml`, `hcpi_outlet_menus.xml`, plus separate files for sub-models. Match the convention so other devs can find things.

## Step 2: a proper form view with a statusbar and smart buttons

Open `views/hcpi_field_report_views.xml` and replace the form view with:

```xml
<record id="view_hcpi_field_report_form" model="ir.ui.view">
    <field name="name">hcpi.field.report.form</field>
    <field name="model">hcpi.field.report</field>
    <field name="arch" type="xml">
        <form>
            <header>
                <button name="action_submit"
                        string="Submit"
                        type="object"
                        class="btn-primary"
                        invisible="state != 'draft'"/>
                <button name="action_approve"
                        string="Approve"
                        type="object"
                        class="btn-primary"
                        invisible="state != 'submitted'"/>
                <button name="action_reject"
                        string="Reject"
                        type="object"
                        invisible="state != 'submitted'"/>
                <button name="action_reset_to_draft"
                        string="Reset to Draft"
                        type="object"
                        invisible="state == 'draft'"/>
                <field name="state" widget="statusbar" statusbar_visible="draft,submitted,approved"/>
            </header>
            <sheet>
                <div class="oe_button_box" name="button_box">
                    <button name="action_view_observations"
                            type="object"
                            class="oe_stat_button"
                            icon="fa-list">
                        <field name="observation_count" widget="statinfo" string="Observations"/>
                    </button>
                </div>
                <div class="oe_title">
                    <h1><field name="name" readonly="1"/></h1>
                </div>
                <group>
                    <group>
                        <field name="date"/>
                        <field name="enumerator_id"/>
                        <field name="region_id"/>
                    </group>
                    <group>
                        <field name="duration_hours"/>
                        <field name="outlets_visited"/>
                        <field name="prices_collected"/>
                        <field name="has_high_severity" readonly="1"/>
                    </group>
                </group>
                <field name="tag_ids" widget="many2many_tags" options="{'color_field': 'color'}"/>
                <notebook>
                    <page string="Observations" name="observations">
                        <field name="observation_ids">
                            <list editable="bottom">
                                <field name="name"/>
                                <field name="outlet_name"/>
                                <field name="severity" widget="badge"
                                       decoration-danger="severity == 'high'"
                                       decoration-warning="severity == 'medium'"
                                       decoration-info="severity == 'low'"/>
                                <field name="needs_action"/>
                            </list>
                            <form>
                                <sheet>
                                    <group>
                                        <field name="name"/>
                                        <field name="outlet_name"/>
                                        <field name="severity"/>
                                        <field name="needs_action"/>
                                    </group>
                                    <field name="notes" placeholder="Details..."/>
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
    <button name="action_submit" ... invisible="state != 'draft'"/>
    ...
    <field name="state" widget="statusbar" statusbar_visible="draft,submitted,approved"/>
</header>
```

The header sits above the sheet. Conventions:

- **Workflow buttons** on the left.
- **State `widget="statusbar"`** on the right — renders the breadcrumb-style progress display.
- **`invisible="<expression>"`** — Odoo 17/18 syntax. The expression is a Python-ish condition evaluated client-side. Old Odoo used `attrs="{'invisible': [(...)]}"`; you'll see both in HCPI code.
- **`statusbar_visible="draft,submitted,approved"`** — show only those three states in the bar (hide "rejected" unless current). Optional polish.

The buttons call Python methods via `name="action_submit" type="object"`. We need to add those methods now. Open `models/hcpi_field_report.py` and add at the bottom of the class:

```python
def action_submit(self):
    for report in self:
        report.state = 'submitted'

def action_approve(self):
    for report in self:
        report.state = 'approved'

def action_reject(self):
    for report in self:
        report.state = 'rejected'

def action_reset_to_draft(self):
    for report in self:
        report.state = 'draft'

def action_view_observations(self):
    self.ensure_one()
    return {
        'type': 'ir.actions.act_window',
        'name': "Observations",
        'res_model': 'hcpi.field.observation',
        'view_mode': 'list,form',
        'domain': [('report_id', '=', self.id)],
        'context': {'default_report_id': self.id},
    }
```

Things to notice:

- **`for report in self`** — always loop, even if buttons are usually clicked on one record at a time. Lists support multi-selection.
- **`self.ensure_one()`** — used in `action_view_observations` because it only makes sense on a single record (it builds a URL). Raises if you have more than one.
- **The dict returned from `action_view_observations`** — that's an Odoo "client action" descriptor. Returning a dict like this from a button tells the UI to navigate. We'll do this pattern again with reports in Part 3.

We'll wire up real validation, sequences, and chatter tracking on these in Part 3 — for now they just flip the state.

### Smart buttons (the box at the top of the form)

```xml
<div class="oe_button_box" name="button_box">
    <button name="action_view_observations"
            type="object"
            class="oe_stat_button"
            icon="fa-list">
        <field name="observation_count" widget="statinfo" string="Observations"/>
    </button>
</div>
```

**Smart buttons** are those large round counters you see on a partner form ("12 Sales Orders", "3 Invoices"). The pattern is:

- `oe_button_box` div in the top-right of the sheet.
- Buttons with class `oe_stat_button` and an icon.
- Inside each button, a `<field widget="statinfo">` showing the count.
- Clicking opens a list of related records.

Even though we already show observations inline in the notebook, the smart button gives a way to focus on just them in their own view — useful when you have many.

### List view inside the One2many — with decorations

```xml
<field name="observation_ids">
    <list editable="bottom">
        ...
        <field name="severity" widget="badge"
               decoration-danger="severity == 'high'"
               decoration-warning="severity == 'medium'"
               decoration-info="severity == 'low'"/>
        ...
    </list>
    <form>
        ...
    </form>
</field>
```

Two things:

1. **`widget="badge"`** + `decoration-*` — renders the field as a coloured pill. The decoration classes (`danger`, `warning`, `info`, `success`, `muted`) map to Bootstrap colours.
2. **An inline `<form>`** in addition to the `<list>` — when the user double-clicks an observation row, this form opens in a modal. Without an inline form, double-click would open a default tiny one.

You can use `decoration-*` on `<list>` rows themselves (`<list decoration-danger="severity == 'high'">`) to highlight whole rows.

## Step 3: install / upgrade and check the form

```bash
python odoo/odoo-bin -c conf/hcpi.conf -u hcpi_field_reports
```

Open a report. The form should now have:

- A status bar at the top with **Draft → Submitted → Approved**.
- Workflow buttons that change based on the current state.
- A smart button counting observations.
- A coloured severity badge in the observations list.

Click **Submit** → the state advances and only **Approve** / **Reject** / **Reset to Draft** show. Click **Approve** → state moves to "Approved".

## Step 4: Kanban view

Kanban is the workflow-board view. Cards grouped by `state` (or any field), drag-to-update.

Add this view *and* update the action to include `kanban`:

```xml
<record id="view_hcpi_field_report_kanban" model="ir.ui.view">
    <field name="name">hcpi.field.report.kanban</field>
    <field name="model">hcpi.field.report</field>
    <field name="arch" type="xml">
        <kanban default_group_by="state" class="o_kanban_small_column">
            <field name="name"/>
            <field name="date"/>
            <field name="enumerator_id"/>
            <field name="region_id"/>
            <field name="outlets_visited"/>
            <field name="prices_collected"/>
            <field name="observation_count"/>
            <field name="has_high_severity"/>
            <field name="tag_ids"/>
            <templates>
                <t t-name="card">
                    <div class="oe_kanban_global_click">
                        <div class="o_kanban_record_title">
                            <strong><field name="name"/></strong>
                            <span class="float-end">
                                <field name="date"/>
                            </span>
                        </div>
                        <div class="o_kanban_record_subtitle">
                            <field name="enumerator_id"/> · <field name="region_id"/>
                        </div>
                        <div class="mt-2">
                            <field name="tag_ids" widget="many2many_tags" options="{'color_field': 'color'}"/>
                        </div>
                        <div class="o_kanban_record_bottom mt-2">
                            <div class="oe_kanban_bottom_left">
                                <i class="fa fa-shopping-basket me-1"/>
                                <field name="outlets_visited"/> outlets
                            </div>
                            <div class="oe_kanban_bottom_right">
                                <span t-if="record.has_high_severity.raw_value" class="badge text-bg-danger">
                                    High severity
                                </span>
                                <span class="badge text-bg-info">
                                    <field name="observation_count"/> obs
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
<record id="action_hcpi_field_report" model="ir.actions.act_window">
    <field name="name">Field Reports</field>
    <field name="res_model">hcpi.field.report</field>
    <field name="view_mode">kanban,list,form,graph,pivot,calendar</field>
</record>
```

The action's `view_mode` lists all the view types this action can switch to. The order also sets the **default** view (kanban first → kanban opens by default).

What's going on inside the kanban:

- **`default_group_by="state"`** — start grouped by state. The columns are the values of `state`. Drag a card across columns to write that value.
- **`<field>` declarations at the top** — every field used inside the template must be declared. The kanban renderer only fetches declared fields.
- **`<templates><t t-name="card">`** — Odoo 18's modern kanban uses the `card` template name. (Older code uses `kanban-box`.) Inside the `<t>` you write standard QWeb-flavoured HTML.
- **`record.has_high_severity.raw_value`** — `record.<fieldname>` inside kanban gives you `.value` (formatted) and `.raw_value` (the actual stored value). For booleans you want `raw_value` to do truthiness checks.

Upgrade and open **Field Reports**. You should land on a kanban board with columns for each state. Drag a card from Draft to Submitted — the state writes.

!!! tip "If kanban doesn't show columns"
    Make sure your `state` field is a `Selection` (it is). Make sure you have records — empty kanban shows the action's "help" placeholder.

## Step 5: Graph view

```xml
<record id="view_hcpi_field_report_graph" model="ir.ui.view">
    <field name="name">hcpi.field.report.graph</field>
    <field name="model">hcpi.field.report</field>
    <field name="arch" type="xml">
        <graph string="Field Reports" type="bar" sample="1">
            <field name="region_id"/>
            <field name="outlets_visited" type="measure"/>
            <field name="prices_collected" type="measure"/>
        </graph>
    </field>
</record>
```

- **`type="bar"`** — also `line`, `pie`.
- **`<field type="measure">`** — what to aggregate. Default aggregation is sum; override with `widget="float_time"` etc. for special displays.
- A field without `type="measure"` is a **group-by axis**.
- **`sample="1"`** — show generated sample data when the real table is empty (handy during demos).

Click the **Graph** view selector in the Field Reports list → see a bar chart of total outlets visited per region.

## Step 6: Pivot view

```xml
<record id="view_hcpi_field_report_pivot" model="ir.ui.view">
    <field name="name">hcpi.field.report.pivot</field>
    <field name="model">hcpi.field.report</field>
    <field name="arch" type="xml">
        <pivot string="Field Reports">
            <field name="region_id" type="row"/>
            <field name="state" type="col"/>
            <field name="outlets_visited" type="measure"/>
        </pivot>
    </field>
</record>
```

Same idea but two-dimensional. Rows = regions, columns = states, cells = sum of outlets visited.

The user can click `+` on a row to expand it (by date, by enumerator, etc.) and the pivot recomputes. **Without writing any code** they can pivot, filter, and group by anything.

## Step 7: Calendar view

```xml
<record id="view_hcpi_field_report_calendar" model="ir.ui.view">
    <field name="name">hcpi.field.report.calendar</field>
    <field name="model">hcpi.field.report</field>
    <field name="arch" type="xml">
        <calendar string="Field Reports"
                  date_start="date"
                  color="enumerator_id"
                  mode="month">
            <field name="enumerator_id"/>
            <field name="region_id"/>
        </calendar>
    </field>
</record>
```

- **`date_start="date"`** — the field that places the record on the calendar.
- **`color="enumerator_id"`** — colour each entry by enumerator. With a M2O, Odoo assigns colours automatically.
- **`mode="month"`** — `day` / `week` / `month` / `year`. The user can switch.

## Step 8: Search view — filters and group bys

```xml
<record id="view_hcpi_field_report_search" model="ir.ui.view">
    <field name="name">hcpi.field.report.search</field>
    <field name="model">hcpi.field.report</field>
    <field name="arch" type="xml">
        <search>
            <!-- Quick text search across these fields -->
            <field name="name"/>
            <field name="enumerator_id"/>
            <field name="region_id"/>
            <field name="tag_ids"/>

            <!-- Predefined filters (one-click) -->
            <filter name="draft" string="Draft" domain="[('state', '=', 'draft')]"/>
            <filter name="submitted" string="Submitted" domain="[('state', '=', 'submitted')]"/>
            <filter name="approved" string="Approved" domain="[('state', '=', 'approved')]"/>
            <separator/>
            <filter name="my_reports" string="My Reports"
                    domain="[('enumerator_id', '=', uid)]"/>
            <filter name="this_month" string="This Month"
                    domain="[('date', '&gt;=', (context_today() + relativedelta(day=1)).strftime('%Y-%m-%d'))]"/>
            <filter name="has_issues" string="With High-Severity Issues"
                    domain="[('has_high_severity', '=', True)]"/>

            <!-- Group-by options -->
            <group expand="0" string="Group By">
                <filter name="group_state" string="State" context="{'group_by': 'state'}"/>
                <filter name="group_enumerator" string="Enumerator" context="{'group_by': 'enumerator_id'}"/>
                <filter name="group_region" string="Region" context="{'group_by': 'region_id'}"/>
                <filter name="group_date" string="Date" context="{'group_by': 'date'}"/>
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
- **`uid`** — a magic variable referring to the current user's id. Use it in domains for "my X" filters.
- **`context_today()` and `relativedelta(...)`** — date helpers Odoo exposes in domain expressions for things like "this month."

Once you reload, the Field Reports list has the new search sidebar. Open it and try:

- Click **My Reports** + **This Month** → narrowed to your own recent work.
- Group By **Region** → list collapses into region groups with counts.
- Group By **Enumerator** then **State** (nested) → two-level grouping.

## Step 9: list view decorations and a server action

Update the list view inside `hcpi_field_report_views.xml`:

```xml
<record id="view_hcpi_field_report_list" model="ir.ui.view">
    <field name="name">hcpi.field.report.list</field>
    <field name="model">hcpi.field.report</field>
    <field name="arch" type="xml">
        <list decoration-info="state == 'submitted'"
              decoration-success="state == 'approved'"
              decoration-muted="state == 'draft'"
              decoration-danger="state == 'rejected'">
            <field name="name"/>
            <field name="date"/>
            <field name="enumerator_id" optional="show"/>
            <field name="region_id" optional="show"/>
            <field name="outlets_visited" sum="Total"/>
            <field name="prices_collected" sum="Total"/>
            <field name="observation_count" optional="show"/>
            <field name="has_high_severity" optional="hide" widget="boolean_toggle"/>
            <field name="state" widget="badge"
                   decoration-info="state == 'submitted'"
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

Add this near the top of `hcpi_field_report_views.xml`:

```xml
<record id="action_server_submit_selected" model="ir.actions.server">
    <field name="name">Submit Selected</field>
    <field name="model_id" ref="model_hcpi_field_report"/>
    <field name="binding_model_id" ref="model_hcpi_field_report"/>
    <field name="binding_view_types">list</field>
    <field name="state">code</field>
    <field name="code">
records.filtered(lambda r: r.state == 'draft').write({'state': 'submitted'})
    </field>
</record>
```

- **`ir.actions.server` + `state='code'`** — server action that runs the Python in `code`.
- **`binding_model_id`** + **`binding_view_types="list"`** — adds this action to the **Actions** drop-down on the list view, only visible there.
- **`records`** — magic variable in server-action code: the selected recordset.
- **`.filtered(lambda r: ...)`** — a recordset method that returns only matching records. Same idea as Python's `filter()`.

After upgrade, select multiple draft reports in the list → **Actions ▾ → Submit Selected** appears.

## Step 10: a printable PDF report

Reports in Odoo are **QWeb** templates rendered to PDF. Add `hcpi_field_report_report.xml` and a folder for reports:

```bash
mkdir -p /opt/hcpi/custom/HCPI/hcpi_field_reports/reports
touch /opt/hcpi/custom/HCPI/hcpi_field_reports/reports/hcpi_field_report_template.xml
```

Add the path to `__manifest__.py` `data`:

```python
'data': [
    'security/ir.model.access.csv',
    'reports/hcpi_field_report_template.xml',
    'views/hcpi_region_views.xml',
    ...
],
```

Then write the report template (`reports/hcpi_field_report_template.xml`):

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <!-- The report action — what makes it appear in the Print menu -->
    <record id="action_report_hcpi_field_report" model="ir.actions.report">
        <field name="name">Field Report</field>
        <field name="model">hcpi.field.report</field>
        <field name="report_type">qweb-pdf</field>
        <field name="report_name">hcpi_field_reports.report_hcpi_field_report_document</field>
        <field name="report_file">hcpi_field_reports.report_hcpi_field_report_document</field>
        <field name="binding_model_id" ref="model_hcpi_field_report"/>
        <field name="binding_type">report</field>
    </record>

    <!-- The QWeb template itself -->
    <template id="report_hcpi_field_report_document">
        <t t-call="web.html_container">
            <t t-foreach="docs" t-as="report">
                <t t-call="web.external_layout">
                    <div class="page">
                        <h2>Field Report — <span t-field="report.name"/></h2>
                        <div class="row mb-3">
                            <div class="col-6">
                                <strong>Date:</strong> <span t-field="report.date"/><br/>
                                <strong>Enumerator:</strong> <span t-field="report.enumerator_id"/><br/>
                                <strong>Region:</strong> <span t-field="report.region_id.complete_name"/>
                            </div>
                            <div class="col-6">
                                <strong>Hours in field:</strong> <span t-field="report.duration_hours"/><br/>
                                <strong>Outlets visited:</strong> <span t-field="report.outlets_visited"/><br/>
                                <strong>Prices collected:</strong> <span t-field="report.prices_collected"/>
                            </div>
                        </div>

                        <h4>Observations</h4>
                        <table class="table table-sm">
                            <thead>
                                <tr>
                                    <th>Description</th>
                                    <th>Outlet</th>
                                    <th>Severity</th>
                                    <th>Action needed?</th>
                                </tr>
                            </thead>
                            <tbody>
                                <tr t-foreach="report.observation_ids" t-as="obs">
                                    <td><span t-field="obs.name"/></td>
                                    <td><span t-field="obs.outlet_name"/></td>
                                    <td>
                                        <span t-attf-class="badge text-bg-#{ {'high': 'danger', 'medium': 'warning', 'low': 'info'}.get(obs.severity, 'secondary') }">
                                            <span t-field="obs.severity"/>
                                        </span>
                                    </td>
                                    <td>
                                        <t t-if="obs.needs_action">Yes</t>
                                        <t t-else="">—</t>
                                    </td>
                                </tr>
                            </tbody>
                        </table>

                        <t t-if="report.notes">
                            <h4>Notes</h4>
                            <p t-field="report.notes"/>
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
| `binding_model_id` + `binding_type="report"` | Wires it under "Print" specifically for `hcpi.field.report`. |
| `web.html_container` | Standard wrapper providing CSS resets and page sizing. |
| `web.external_layout` | Adds header (company name/logo) and footer. Use `web.internal_layout` to skip. |
| `t-foreach`, `t-as` | Loop over records (`docs` is the recordset passed in). |
| `t-field="report.name"` | Renders the field value with formatting (dates, monetary, etc.). |
| `t-attf-class="...#{ expr }..."` | String formatting inside an attribute (the `#{}` is QWeb syntax). |
| `t-if` / `t-else` | Conditionals. |

After upgrade, open a report → click **Print ▾ → Field Report**. A PDF downloads with the formatted report.

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

class FieldSeverityBadge extends Component {
    static template = "hcpi_field_reports.SeverityBadge";
    get colorClass() {
        return {
            low: "text-bg-info",
            medium: "text-bg-warning",
            high: "text-bg-danger",
        }[this.props.record.data.severity] || "text-bg-secondary";
    }
}

registry.category("fields").add("severity_badge", { component: FieldSeverityBadge });
```

The takeaway: **most Odoo development never touches OWL.** You build models, write views in XML, and the framework gives you a rich UI for free. OWL is the escape hatch for the 5% case.

## Exercises

1. **Add a Gantt-like calendar view** that uses `date_start="date"` and adds a duration in days (compute a `date_end` from `date + duration_hours/8`). Switch the action to allow `calendar,gantt`.

2. **Custom group-by**: add a group-by filter for "Has High Severity" — toggle the search to group reports into Yes/No.

3. **Server action that uses `mapped`**: write an `ir.actions.server` that on selected reports, prints (via `raise UserError`) a summary like `"Selected: 5 reports, 23 outlets, 142 prices"`. Hint: `records.mapped('outlets_visited')` returns a list; sum it.

4. **Override the report layout**: add a logo by creating a `report_assets` block or override `web.external_layout`. (Stretch — comes up rarely but worth knowing it's possible.)

5. **Add a `priority` field**: `Selection([('0', 'Normal'), ('1', 'Important'), ('2', 'Urgent')])` with `widget="priority"` on the form (it renders as stars). Add it to the kanban card top-right corner.

## What you learned

After Part 2 you understand:

- **Every view type** (List, Form, Search, Kanban, Graph, Pivot, Calendar) and when to reach for each.
- **Form composition**: header + statusbar + smart buttons + sheet + notebook + pages.
- **Workflow buttons** that call Python methods via `name="..."` + `type="object"`.
- **Inline One2many editing** in lists, plus an inline form for the modal editor.
- **Search views**: domain filters, "my X" with `uid`, date helpers, separators, group-by.
- **List decorations**, sums, optional columns, badges.
- **Server actions** with `binding_*` to expose them in the UI.
- **QWeb PDF reports**: `ir.actions.report`, `web.external_layout`, `t-foreach`, `t-field`, `t-if`.
- **What OWL is** and when you'd need to learn it deeper.

## What's next

➡️ **[Part 3: Security & Polish](part3-security.md)** — close the module. Define collector and supervisor groups, restrict access at the model and record level, add a sequence for auto-naming, wire `mail.thread` for chatter and audit trail, and ship a production-ready module.
