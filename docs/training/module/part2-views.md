# Building an HCPI Module — Part 2: Views & UX

You've got a module with data flowing in and out. Now we make it pleasant to use.

## A quick tour of the view types

| View | XML tag | When to use it |
|---|---|---|
| **List** (a.k.a. Tree) | `<list>` | Tabular — many records at once. The default. |
| **Form** | `<form>` | One record at a time, fields laid out for editing. |
| **Search** | `<search>` | The filter sidebar with predefined filters + group bys. |
| **Kanban** | `<kanban>` | Cards grouped by a field (typically `state`). Workflow boards. |
| **Graph** | `<graph>` | Bar / line / pie charts over aggregated data. |
| **Pivot** | `<pivot>` | Spreadsheet-style cross-tabs. |
| **Calendar** | `<calendar>` | Records placed on a date. Day/week/month switcher. |

We'll build all seven. Other view types (Activity, Gantt) follow the same pattern.

## Step 1: a proper form with a statusbar and smart button

Open `views/views.xml` and replace the proposal form view with:

```xml
<record id="view_hcpi_outlet_proposal_form" model="ir.ui.view">
    <field name="name">hcpi.outlet.proposal.form</field>
    <field name="model">hcpi.outlet.proposal</field>
    <field name="arch" type="xml">
        <form>
            <header>
                <button name="action_approve"
                        string="Approve"
                        type="object"
                        class="btn-primary"
                        invisible="state != 'draft'"/>
                <button name="action_retire"
                        string="Retire"
                        type="object"
                        invisible="state != 'active'"/>
                <button name="action_reset_to_draft"
                        string="Reset to Draft"
                        type="object"
                        invisible="state == 'draft'"/>
                <field name="state" widget="statusbar"
                       statusbar_visible="draft,active,retired"/>
            </header>
            <sheet>
                <div class="oe_button_box" name="button_box">
                    <button name="action_view_visits"
                            type="object"
                            class="oe_stat_button"
                            icon="fa-map-marker">
                        <field name="visit_count" widget="statinfo" string="Visits"/>
                    </button>
                </div>
                <div class="oe_title">
                    <label for="outlet_name"/>
                    <h1><field name="outlet_name" placeholder="e.g. Kikuubo Wholesale Market"/></h1>
                    <span class="text-muted"><field name="name" readonly="1"/></span>
                </div>
                <group>
                    <group string="Classification">
                        <field name="outlet_type"/>
                        <field name="region"/>
                        <field name="proposed_by"/>
                        <field name="opening_date"/>
                    </group>
                    <group string="Location &amp; Contact">
                        <field name="contact_name"/>
                        <field name="contact_phone"/>
                        <label for="latitude" string="GPS"/>
                        <div>
                            <field name="latitude" class="oe_inline"/>,
                            <field name="longitude" class="oe_inline"/>
                        </div>
                    </group>
                </group>
                <field name="assigned_user_ids" widget="many2many_avatar_user"/>
                <notebook>
                    <page string="Address" name="address">
                        <field name="address" placeholder="Street, landmark, neighbourhood..."/>
                    </page>
                    <page string="Visits" name="visits">
                        <field name="visit_ids">
                            <list editable="bottom">
                                <field name="date"/>
                                <field name="name"/>
                                <field name="visitor_id"/>
                                <field name="result" widget="badge"
                                       decoration-danger="result == 'closed'"
                                       decoration-warning="result == 'partial'"
                                       decoration-success="result == 'open'"/>
                            </list>
                            <form>
                                <sheet>
                                    <group>
                                        <field name="name"/>
                                        <field name="date"/>
                                        <field name="visitor_id"/>
                                        <field name="result"/>
                                    </group>
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

Walk through the new pieces section by section.

### `<header>` and the statusbar

```xml
<header>
    <button name="action_approve" ... invisible="state != 'draft'"/>
    ...
    <field name="state" widget="statusbar" statusbar_visible="draft,active,retired"/>
</header>
```

The header sits above the sheet. Conventions:

- **Workflow buttons** on the left.
- **`<field name="state" widget="statusbar">`** on the right — renders the breadcrumb-style progress display.
- **`invisible="<expression>"`** — Odoo 17/18 syntax. The expression is a Python-ish condition evaluated client-side.
- **`statusbar_visible="draft,active,retired"`** — show those three stages in the bar.

### Smart button

```xml
<div class="oe_button_box" name="button_box">
    <button name="action_view_visits"
            type="object"
            class="oe_stat_button"
            icon="fa-map-marker">
        <field name="visit_count" widget="statinfo" string="Visits"/>
    </button>
</div>
```

**Smart buttons** are those large counters you see on a partner form ("12 Sales Orders", "3 Invoices"). The pattern is `oe_button_box` div + `oe_stat_button` class + a Font Awesome icon + an inner `<field widget="statinfo">` showing the count. Clicking opens a list of related records.

### Inline `<form>` inside the One2many

The notebook's Visits page already had a `<list editable="bottom">` from Part 1. We've added a `<form>` alongside it — when the user double-clicks a visit row, this form opens in a modal. Without an inline form, double-click would open a tiny default one.

The `widget="badge" decoration-*` on `result` renders it as a coloured pill — `danger`, `warning`, `info`, `success`, `muted` map to Bootstrap colours.

### The team chip widget

```xml
<field name="assigned_user_ids" widget="many2many_avatar_user"/>
```

`many2many_avatar_user` renders the user M2M as little avatar circles with hover tooltips — nicer than a multi-select dropdown. Odoo ships several Many2many widgets: `many2many_tags` for generic chips, `many2many_avatar_user` for users, `many2many_checkboxes` for inline toggles.

## Step 2: add the button methods

The header buttons call Python via `name="action_..."` `type="object"`. Open `models/hcpi_outlet_proposal.py` and add at the bottom of the class:

```python
def action_approve(self):
    for proposal in self:
        proposal.state = 'active'

def action_retire(self):
    for proposal in self:
        proposal.state = 'retired'

def action_reset_to_draft(self):
    for proposal in self:
        proposal.state = 'draft'

def action_view_visits(self):
    self.ensure_one()
    return {
        'type': 'ir.actions.act_window',
        'name': "Visits",
        'res_model': 'hcpi.outlet.visit',
        'view_mode': 'list,form',
        'domain': [('proposal_id', '=', self.id)],
        'context': {'default_proposal_id': self.id},
    }
```

Things to notice:

- **`for proposal in self`** — always loop, even if a button is usually clicked on one record at a time. Lists support multi-selection.
- **`self.ensure_one()`** — used in `action_view_visits` because it only makes sense on a single record. Raises if you have more than one.
- **The dict returned from `action_view_visits`** — that's an Odoo "client action" descriptor. Returning a dict like this from a button tells the UI to navigate. The `default_proposal_id` in `context` pre-fills that field when creating a new visit from the resulting view.

Hardened validation (only managers can approve, can't approve without a visit, etc.) comes in Part 3.

## Step 3: upgrade and check the form

```bash
python odoo/odoo-bin -c conf/hcpi.conf -u hcpi_outlet_onboarding
```

Open a proposal:

- A status bar at the top with **Draft → Active → Retired**.
- Workflow buttons that change based on the current state.
- A smart button counting visits.
- A coloured result badge in the visit list.
- Assigned team rendered as avatar circles.

Click **Approve** → state advances to Active and the **Retire** button replaces it. Click **Reset to Draft** → back to Draft.

## Step 4: Kanban view

Kanban is the workflow-board view. Cards grouped by `state` (or any field), drag-to-update. Add this view to `views.xml`:

```xml
<record id="view_hcpi_outlet_proposal_kanban" model="ir.ui.view">
    <field name="name">hcpi.outlet.proposal.kanban</field>
    <field name="model">hcpi.outlet.proposal</field>
    <field name="arch" type="xml">
        <kanban default_group_by="state" class="o_kanban_small_column">
            <field name="outlet_name"/>
            <field name="outlet_type"/>
            <field name="region"/>
            <field name="proposed_by"/>
            <field name="visit_count"/>
            <field name="assigned_user_ids"/>
            <templates>
                <t t-name="card">
                    <div class="oe_kanban_global_click">
                        <div class="o_kanban_record_title">
                            <strong><field name="outlet_name"/></strong>
                        </div>
                        <div class="o_kanban_record_subtitle">
                            <field name="region"/> · <field name="outlet_type"/>
                        </div>
                        <div class="mt-2">
                            <field name="assigned_user_ids" widget="many2many_avatar_user"/>
                        </div>
                        <div class="o_kanban_record_bottom mt-2">
                            <div class="oe_kanban_bottom_left">
                                <i class="fa fa-user me-1"/>
                                <field name="proposed_by"/>
                            </div>
                            <div class="oe_kanban_bottom_right">
                                <span class="badge text-bg-info">
                                    <i class="fa fa-map-marker me-1"/>
                                    <field name="visit_count"/>
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

And update the action to expose kanban (and the other views we're about to add):

```xml
<record id="action_hcpi_outlet_proposal" model="ir.actions.act_window">
    <field name="name">Outlet Proposals</field>
    <field name="res_model">hcpi.outlet.proposal</field>
    <field name="view_mode">kanban,list,form,graph,pivot,calendar</field>
</record>
```

The action's `view_mode` lists every view type this action can switch to. The order sets the **default** (kanban first → kanban opens by default).

What's going on inside the kanban:

- **`default_group_by="state"`** — start grouped by state. The columns are the values of `state`. Drag a card across columns to write that value.
- **`<field>` declarations at the top** — every field used inside the template must be declared. The kanban renderer only fetches declared fields.
- **`<templates><t t-name="card">`** — Odoo 18's modern kanban uses the `card` template name. Inside the `<t>` you write standard QWeb-flavoured HTML.

Upgrade and open **Outlet Onboarding → Proposals**. You should land on a kanban board with three columns (Draft, Active, Retired). Drag a card from Draft to Active — the state writes.

## Step 5: Graph view

```xml
<record id="view_hcpi_outlet_proposal_graph" model="ir.ui.view">
    <field name="name">hcpi.outlet.proposal.graph</field>
    <field name="model">hcpi.outlet.proposal</field>
    <field name="arch" type="xml">
        <graph string="Outlet Proposals" type="bar" sample="1">
            <field name="outlet_type"/>
            <field name="visit_count" type="measure"/>
        </graph>
    </field>
</record>
```

- **`type="bar"`** — also `line`, `pie`.
- **`<field type="measure">`** — what to aggregate. Default aggregation is sum.
- A field without `type="measure"` is a **group-by axis**.
- **`sample="1"`** — show generated sample data when the real table is empty (handy during demos).

Click the **Graph** view selector in the Proposals list → bar chart of total visits per outlet type.

## Step 6: Pivot view

```xml
<record id="view_hcpi_outlet_proposal_pivot" model="ir.ui.view">
    <field name="name">hcpi.outlet.proposal.pivot</field>
    <field name="model">hcpi.outlet.proposal</field>
    <field name="arch" type="xml">
        <pivot string="Outlet Proposals">
            <field name="outlet_type" type="row"/>
            <field name="state" type="col"/>
            <field name="visit_count" type="measure"/>
        </pivot>
    </field>
</record>
```

Two-dimensional aggregation. Rows = outlet types, columns = states, cells = sum of visits. The user can click `+` on a row to expand it and the pivot recomputes — they can pivot, filter, and group by anything **without writing any code**.

## Step 7: Calendar view

```xml
<record id="view_hcpi_outlet_proposal_calendar" model="ir.ui.view">
    <field name="name">hcpi.outlet.proposal.calendar</field>
    <field name="model">hcpi.outlet.proposal</field>
    <field name="arch" type="xml">
        <calendar string="Outlet Proposals"
                  date_start="opening_date"
                  color="outlet_type"
                  mode="month">
            <field name="outlet_name"/>
            <field name="outlet_type"/>
        </calendar>
    </field>
</record>
```

- **`date_start="opening_date"`** — the field that places the record on the calendar. Outlets show up on their opening date.
- **`color="outlet_type"`** — colour each entry by outlet type. Odoo assigns colours automatically.
- **`mode="month"`** — `day` / `week` / `month` / `year`. The user can switch.

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
            <field name="region"/>
            <field name="proposed_by"/>

            <!-- Predefined filters -->
            <filter name="draft" string="Draft" domain="[('state', '=', 'draft')]"/>
            <filter name="active" string="Active" domain="[('state', '=', 'active')]"/>
            <filter name="retired" string="Retired" domain="[('state', '=', 'retired')]"/>
            <separator/>
            <filter name="mine" string="My Proposals"
                    domain="[('proposed_by', '=', uid)]"/>
            <filter name="no_visits" string="No Visits Yet"
                    domain="[('visit_count', '=', 0)]"/>

            <!-- Group-by options -->
            <group expand="0" string="Group By">
                <filter name="group_state" string="State"
                        context="{'group_by': 'state'}"/>
                <filter name="group_type" string="Outlet Type"
                        context="{'group_by': 'outlet_type'}"/>
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
- **`uid`** — magic variable referring to the current user's id. Use it for "my X" filters.

Once you reload, try:

- Click **My Proposals** → narrowed to your own work.
- Group By **State** → list collapses into state groups with counts.
- Group By **State** then **Outlet Type** (nested) → two-level grouping.

## Step 9: list view decorations

Replace the proposal list view:

```xml
<record id="view_hcpi_outlet_proposal_list" model="ir.ui.view">
    <field name="name">hcpi.outlet.proposal.list</field>
    <field name="model">hcpi.outlet.proposal</field>
    <field name="arch" type="xml">
        <list decoration-info="state == 'draft'"
              decoration-success="state == 'active'"
              decoration-muted="state == 'retired'">
            <field name="name"/>
            <field name="outlet_name"/>
            <field name="outlet_type" optional="show"/>
            <field name="region" optional="show"/>
            <field name="proposed_by" optional="show"/>
            <field name="visit_count" optional="show" sum="Total"/>
            <field name="state" widget="badge"
                   decoration-info="state == 'draft'"
                   decoration-success="state == 'active'"
                   decoration-muted="state == 'retired'"/>
        </list>
    </field>
</record>
```

- **Row-level decorations** colour the whole row by state.
- **`sum="Total"`** on numeric columns shows a sum row at the bottom (or below the current group when grouped).
- **`optional="show"` / `optional="hide"`** — column can be toggled by the user from the column-picker menu.

## Exercises

1. **Visit calendar.** Add a Calendar view to `hcpi.outlet.visit` with `date_start="date"` and `color="visitor_id"`. Expose it via a new action and menu. Each visit shows on the day it happened.

2. **Wire up `priority`.** You added `priority` in Part 1's exercises. Display it on the form with `widget="priority"` (renders as stars) and in the kanban card top-right corner.

3. **Server action from the list.** Add an `ir.actions.server` with `binding_model_id` and `binding_view_types="list"` that runs Python on the selected records to set their state to `active`. Hint: the magic variable inside `<field name="code">` is `records`.

4. **QWeb PDF report.** Add an `ir.actions.report` with `report_type="qweb-pdf"` and a `<template>` that renders the proposal as a printable dossier. Use `web.external_layout` for header/footer. Bind it to the form via `binding_model_id` so it appears in the Print menu.

5. **Custom group-by.** Add a group-by filter for "Has Visits" — true if `visit_count > 0`, false otherwise. Hint: a computed boolean field plus a `context="{'group_by': '...'}"` filter.

## What you learned

After Part 2 you understand:

- **Every main view type** (List, Form, Search, Kanban, Graph, Pivot, Calendar) and when to reach for each.
- **Form composition**: header + statusbar + smart buttons + sheet + notebook + pages.
- **Workflow buttons** that call Python methods via `name="..."` + `type="object"`, and returning a window-action dict to navigate.
- **Inline One2many editing** in lists, plus an inline form for the modal editor.
- **Search views**: domain filters, "my X" with `uid`, separators, group-by.
- **List decorations**, sums, optional columns, badges.
- **Several Many2many widgets** — `many2many_tags`, `many2many_avatar_user`.

## What's next

➡️ **[Part 3: Security & Polish](part3-security.md)** — close the module. Add a Manager role, auto-numbered references, mail.thread chatter and audit trail, Python validation, and harden the workflow buttons.
