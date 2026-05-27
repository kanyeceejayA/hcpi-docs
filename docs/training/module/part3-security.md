# Building HCPI Field Reports — Part 3: Security & Polish

**Duration:** half a day (about 4 hours).
**Covers:** user groups, model-level access (`ir.model.access.csv`), record-level rules (`ir.rule`), sequences for auto-numbering, `@api.constrains`, `mail.thread` chatter and audit trail, button workflows with validation, the module's final polish.

You have a working module with data and good UX. Part 3 makes it production-ready: only the right people can see and change the right things, every change is auditable, and the workflow refuses to let you skip steps.

!!! info "Before you start"
    Parts 1 and 2 are complete. You have `hcpi_field_reports` installed with the form, kanban, graph, pivot, calendar, search, and PDF report all working.

## What "security" means in Odoo

Two distinct layers, each ultimately stored as rows in built-in tables (just like views, actions, and menus — recall the picture from [Odoo Basics](../../understanding-the-codebase/odoo-basics.md)):

| Layer | Stored in | What it controls |
|---|---|---|
| **Groups** | `res.groups` | Who is in which "role." Users belong to groups. |
| **Model access (ACL)** | `ir.model.access` | Per-group, per-model: can the group **C**reate / **R**ead / **U**pdate / **D**elete this model at all? |
| **Record rules** | `ir.rule` | Per-group, per-model: which **rows** can the group see/edit? Implemented as a domain auto-appended to every query. |

The mental model: **ACL is the door, record rules are the floor plan.** ACL says "you may enter the room." Record rules say "and within that room you can only see your own desk."

## Step 1: define the security groups

Create `security/hcpi_field_reports_security.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <!-- Category — a header in the user permissions UI -->
    <record id="module_category_hcpi_field_reports" model="ir.module.category">
        <field name="name">HCPI Field Reports</field>
        <field name="description">Manage field-activity reports</field>
        <field name="sequence">25</field>
    </record>

    <!-- Groups, ordered from least to most privileged -->
    <record id="group_field_reports_collector" model="res.groups">
        <field name="name">Collector</field>
        <field name="category_id" ref="module_category_hcpi_field_reports"/>
        <field name="comment">Submit their own field reports.</field>
    </record>

    <record id="group_field_reports_supervisor" model="res.groups">
        <field name="name">Supervisor</field>
        <field name="category_id" ref="module_category_hcpi_field_reports"/>
        <field name="implied_ids" eval="[(4, ref('group_field_reports_collector'))]"/>
        <field name="comment">Review and approve any collector's reports.</field>
    </record>

    <record id="group_field_reports_manager" model="res.groups">
        <field name="name">Manager</field>
        <field name="category_id" ref="module_category_hcpi_field_reports"/>
        <field name="implied_ids" eval="[(4, ref('group_field_reports_supervisor'))]"/>
        <field name="comment">Full control, including configuration.</field>
    </record>

</odoo>
```

Walk through:

- **`ir.module.category`** — a UI grouping in **Settings → Users → User → Access Rights**. Without a category, your three groups show up as a dropdown rather than radio buttons grouped under "HCPI Field Reports".
- **`res.groups`** — the actual group records.
- **`implied_ids = [(4, ref('group_field_reports_collector'))]`** — "anyone in Supervisor is automatically also in Collector." `(4, ref('...'))` is the link-an-existing-record command tuple you learned in [Part 1](part1-models.md#the-command-tuples-for-relational-writes).
- The triangle relation **Manager → Supervisor → Collector** means a Manager has all the rights of the levels below. This is the standard Odoo pattern.

Add the file to `__manifest__.py` `data` (security files load **before** views — security records must exist before views reference them):

```python
'data': [
    'security/hcpi_field_reports_security.xml',     # ← new, FIRST
    'security/ir.model.access.csv',
    'reports/hcpi_field_report_template.xml',
    'views/hcpi_region_views.xml',
    ...
],
```

## Step 2: replace the open ACL with a proper matrix

Open `security/ir.model.access.csv` and replace its content:

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink

access_hcpi_region_collector,hcpi.region collector read,model_hcpi_region,hcpi_field_reports.group_field_reports_collector,1,0,0,0
access_hcpi_region_manager,hcpi.region manager all,model_hcpi_region,hcpi_field_reports.group_field_reports_manager,1,1,1,1

access_hcpi_field_tag_collector,hcpi.field.tag collector read,model_hcpi_field_tag,hcpi_field_reports.group_field_reports_collector,1,0,0,0
access_hcpi_field_tag_manager,hcpi.field.tag manager all,model_hcpi_field_tag,hcpi_field_reports.group_field_reports_manager,1,1,1,1

access_hcpi_field_observation_collector,hcpi.field.observation collector all,model_hcpi_field_observation,hcpi_field_reports.group_field_reports_collector,1,1,1,1
access_hcpi_field_observation_supervisor,hcpi.field.observation supervisor all,model_hcpi_field_observation,hcpi_field_reports.group_field_reports_supervisor,1,1,1,1

access_hcpi_field_report_collector,hcpi.field.report collector,model_hcpi_field_report,hcpi_field_reports.group_field_reports_collector,1,1,1,0
access_hcpi_field_report_supervisor,hcpi.field.report supervisor,model_hcpi_field_report,hcpi_field_reports.group_field_reports_supervisor,1,1,1,0
access_hcpi_field_report_manager,hcpi.field.report manager,model_hcpi_field_report,hcpi_field_reports.group_field_reports_manager,1,1,1,1
```

Reading row by row:

| Model | Group | C | R | U | D | Why |
|---|---|---|---|---|---|---|
| `hcpi.region` | Collector | 0 | 1 | 0 | 0 | They need to *pick* a region but shouldn't be able to edit the master data. |
| `hcpi.region` | Manager | 1 | 1 | 1 | 1 | Master data administration. |
| `hcpi.field.tag` | Collector | 0 | 1 | 0 | 0 | Pick tags, don't create them ad-hoc. |
| `hcpi.field.tag` | Manager | 1 | 1 | 1 | 1 | Manage tag list. |
| `hcpi.field.observation` | Collector | 1 | 1 | 1 | 1 | They author observations on their own reports. |
| `hcpi.field.report` | Collector | 1 | 1 | 1 | 0 | Create and edit, but **no delete** — keep an audit trail. |
| `hcpi.field.report` | Manager | 1 | 1 | 1 | 1 | Manager can delete (e.g., garbage cleanup). |

**Key principle:** ACL is binary at the model level. Use ACL for the rough "can this role even touch this model?" question. For nuance (this row but not that row) use **record rules** in the next step.

## Step 3: record rules — "you only see your own reports"

A collector should see only their own reports; a supervisor sees everyone's. We codify that with `ir.rule`.

Add to `security/hcpi_field_reports_security.xml`, after the groups:

```xml
<!-- A collector can only see their OWN reports -->
<record id="rule_field_report_own" model="ir.rule">
    <field name="name">Field Report: own reports only</field>
    <field name="model_id" ref="model_hcpi_field_report"/>
    <field name="domain_force">[('enumerator_id', '=', user.id)]</field>
    <field name="groups" eval="[(4, ref('group_field_reports_collector'))]"/>
    <field name="perm_read" eval="True"/>
    <field name="perm_write" eval="True"/>
    <field name="perm_create" eval="True"/>
    <field name="perm_unlink" eval="True"/>
</record>

<!-- Supervisor sees ALL reports (override the more restrictive rule above) -->
<record id="rule_field_report_supervisor_all" model="ir.rule">
    <field name="name">Field Report: supervisor sees all</field>
    <field name="model_id" ref="model_hcpi_field_report"/>
    <field name="domain_force">[(1, '=', 1)]</field>
    <field name="groups" eval="[(4, ref('group_field_reports_supervisor'))]"/>
</record>

<!-- A collector can only see observations on their OWN reports -->
<record id="rule_field_observation_own" model="ir.rule">
    <field name="name">Field Observation: own reports only</field>
    <field name="model_id" ref="model_hcpi_field_observation"/>
    <field name="domain_force">[('report_id.enumerator_id', '=', user.id)]</field>
    <field name="groups" eval="[(4, ref('group_field_reports_collector'))]"/>
</record>

<record id="rule_field_observation_supervisor_all" model="ir.rule">
    <field name="name">Field Observation: supervisor sees all</field>
    <field name="model_id" ref="model_hcpi_field_observation"/>
    <field name="domain_force">[(1, '=', 1)]</field>
    <field name="groups" eval="[(4, ref('group_field_reports_supervisor'))]"/>
</record>
```

Reading this:

- **`domain_force`** — a domain auto-appended to every search. The user never sees rows that don't match.
- **`user.id`** — magic variable in record rules: the currently logged-in user's id.
- **`(1, '=', 1)`** — the "match everything" domain. Used to give a higher-privilege group full visibility.
- **`perm_read` / `perm_write` / `perm_create` / `perm_unlink`** — which operations the rule applies to. If omitted, the rule applies to **all four**. We're explicit on the first rule for clarity.

### How rules combine

This trips everyone up at first:

**Across groups: rules are OR'd.** If a user is in both Collector and Supervisor, they see the union of what each rule allows. That's why the supervisor rule with `(1, '=', 1)` effectively grants full visibility.

**Within one group, rules are AND'd.** If a single group has two rules, both must match.

So for our triangle (Supervisor implies Collector), a supervisor user has two applicable rules. The Supervisor's `(1, '=', 1)` OR'd with the Collector's `enumerator_id = user.id` simplifies to `True`. They see everything. 

### Global rules vs group-specific rules

A rule with `global=True` (no groups) applies to **everyone** and is AND'd with group rules. We don't need global rules here — the rule applies only to the listed groups via `groups`.

## Step 4: install groups onto the admin

Restart Odoo with the upgrade flag:

```bash
python odoo/odoo-bin -c conf/hcpi.conf -u hcpi_field_reports
```

In the UI, **Settings → Users & Companies → Users → Administrator → Access Rights** tab. Under "HCPI Field Reports" you should see your three radio buttons:

- Collector
- Supervisor
- Manager (selected automatically — admin gets everything)

If you want to test the rules properly:

1. Create a second user **collector1** with email + name. On their **Access Rights** tab, give them "HCPI Field Reports → Collector" (and nothing else).
2. Set their password (top of user form → ⋮ → **Change Password**).
3. Open a private/incognito window and log in as `collector1`.
4. Open **Field Reports** → you see **only your own reports** (which is none yet, until you create one as `collector1`).

Try filing one report as `collector1`. Then log back in as admin — you see both your own admin report and the collector1 one. Rules working.

## Step 5: a sequence for auto-numbering

Replace `name="New"` with proper references like `FR/2026/0001`.

Add a sequence record. Create `data/hcpi_field_report_data.xml`:

```bash
mkdir -p /opt/hcpi/custom/HCPI/hcpi_field_reports/data
touch /opt/hcpi/custom/HCPI/hcpi_field_reports/data/hcpi_field_report_data.xml
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="seq_hcpi_field_report" model="ir.sequence">
        <field name="name">HCPI Field Report</field>
        <field name="code">hcpi.field.report</field>
        <field name="prefix">FR/%(year)s/</field>
        <field name="padding">4</field>
        <field name="company_id" eval="False"/>
    </record>
</odoo>
```

- **`code`** — the handle used in Python (`env['ir.sequence'].next_by_code('hcpi.field.report')`).
- **`prefix`** — `%(year)s` is auto-replaced with the current year. Other tokens: `%(month)s`, `%(day)s`, etc.
- **`padding=4`** — pad the counter to 4 digits with leading zeros: `0001`, `0002`, …
- **`company_id` eval="False"`** — no per-company numbering. (For multi-company instances you might want a different value here.)

Add to `__manifest__.py`:

```python
'data': [
    'security/hcpi_field_reports_security.xml',
    'security/ir.model.access.csv',
    'data/hcpi_field_report_data.xml',          # ← new
    'reports/hcpi_field_report_template.xml',
    'views/hcpi_region_views.xml',
    ...
],
```

Now override `create()` on the report model so newly-created reports pull from the sequence. In `models/hcpi_field_report.py`, add at the bottom of the class:

```python
from odoo import api, fields, models


class HcpiFieldReport(models.Model):
    # ... (existing fields and methods) ...

    @api.model_create_multi
    def create(self, vals_list):
        for vals in vals_list:
            if vals.get('name', "New") == "New":
                vals['name'] = self.env['ir.sequence'].next_by_code('hcpi.field.report') or "New"
        return super().create(vals_list)
```

Note: don't add the `from odoo import...` line again — it's already at the top of the file. Just add the `create` method inside the class.

Reading the override:

- **`@api.model_create_multi`** — Odoo 17+ batch-create decorator. `vals_list` is a list of dicts (one per new record). This signature replaces the older single-dict `create()`.
- **The `"New"` guard** — only generate a sequence number if the caller didn't already pass a name (so duplication / import flows keep their names).
- **`super().create(vals_list)`** — chain to the parent (the framework's default behaviour) so the records actually get created.

After upgrade (`-u hcpi_field_reports`) and creating a new report, the reference should read `FR/2026/0001`.

## Step 6: mail.thread — chatter and audit trail

The chatter is that comments/log section at the bottom of forms in Odoo. It's not just chat — it's also where field changes are logged when fields are marked `tracking=True`. You already added `tracking=True` to `state` in Part 1; now we wire up the mixin to make it visible.

In `models/hcpi_field_report.py`, change the class header:

```python
class HcpiFieldReport(models.Model):
    _name = 'hcpi.field.report'
    _description = "Field Report"
    _inherit = ['mail.thread', 'mail.activity.mixin']
    _order = 'date desc, id desc'
```

That's all the Python side needs. `mail.thread` adds:

- A `message_ids` One2many to all chatter messages.
- A `message_follower_ids` for follower management.
- Auto-tracking of fields with `tracking=True`.

`mail.activity.mixin` adds:

- `activity_ids` for scheduled activities ("call this enumerator next Tuesday").

Then add the chatter to the form view. At the bottom of the form's `<sheet>`, after closing `</sheet>` but inside `</form>`:

```xml
        </sheet>
        <chatter/>
    </form>
```

The `<chatter/>` tag is Odoo 17/18 shorthand. Older code uses an expanded `<div class="oe_chatter">` block — same effect.

Upgrade. Open a report. At the bottom of the form a chatter panel now appears:

- **Send message** — adds a comment.
- **Log note** — internal-only note.
- **Followers** — manage who gets notified.
- Above the input, the **log feed** — and you'll see the `state` change you made earlier already recorded as **State: Draft → Submitted**.

Make a few state changes via the workflow buttons. Each one gets logged automatically.

### Tracking more fields

Pick any field where you want history. Add `tracking=True`:

```python
outlets_visited = fields.Integer(default=0, tracking=True)
prices_collected = fields.Integer(default=0, tracking=True)
region_id = fields.Many2one('hcpi.region', string="Region", required=True, index=True, tracking=True)
```

Upgrade — every change to these fields shows up in the chatter as `Field: old → new`.

## Step 7: validation with `@api.constrains`

Right now nothing prevents an enumerator from submitting a report with negative outlet counts or a date in the future. Add Python constraints.

In `models/hcpi_field_report.py`, add inside the class:

```python
from odoo.exceptions import ValidationError    # add this import at the top of the file

# ... inside the class ...

@api.constrains('outlets_visited', 'prices_collected', 'duration_hours')
def _check_non_negative(self):
    for report in self:
        if report.outlets_visited < 0:
            raise ValidationError("Outlets visited cannot be negative.")
        if report.prices_collected < 0:
            raise ValidationError("Prices collected cannot be negative.")
        if report.duration_hours < 0:
            raise ValidationError("Duration cannot be negative.")

@api.constrains('date')
def _check_date_not_future(self):
    for report in self:
        if report.date and report.date > fields.Date.context_today(report):
            raise ValidationError("Field reports cannot be dated in the future.")
```

`@api.constrains` runs on every save. Different from `@api.onchange` (which runs in the UI as the user types but doesn't block save — it's for hints) and from `_sql_constraints` (database-level, faster but limited to what SQL can express).

Try saving a report with `outlets_visited = -1` → red error toast appears, save blocked.

## Step 8: harden the workflow buttons

Right now `action_submit` lets anyone submit any draft. Add some sanity checks plus a confirmation dialog. Replace the action methods in `models/hcpi_field_report.py`:

```python
def action_submit(self):
    for report in self:
        if report.state != 'draft':
            raise UserError("Only draft reports can be submitted.")
        if not report.observation_ids and report.outlets_visited == 0:
            raise UserError("Add at least one observation or record some outlet visits before submitting.")
        report.state = 'submitted'
        report.message_post(body="Report submitted for review.")

def action_approve(self):
    self._require_supervisor()
    for report in self:
        if report.state != 'submitted':
            raise UserError("Only submitted reports can be approved.")
        report.state = 'approved'
        report.message_post(body="Report approved.")

def action_reject(self):
    self._require_supervisor()
    for report in self:
        if report.state != 'submitted':
            raise UserError("Only submitted reports can be rejected.")
        report.state = 'rejected'
        report.message_post(body="Report rejected.")

def action_reset_to_draft(self):
    self._require_supervisor()
    for report in self:
        report.state = 'draft'
        report.message_post(body="Report reset to draft.")

def _require_supervisor(self):
    if not self.env.user.has_group('hcpi_field_reports.group_field_reports_supervisor'):
        raise UserError("Only supervisors can perform this action.")
```

Don't forget the import at the top of the file:

```python
from odoo.exceptions import UserError, ValidationError
```

Reading the changes:

- **`raise UserError(...)`** — friendly dialog to the user. Different from `ValidationError` mainly by convention; both stop the operation and surface their message.
- **`has_group('<xml-id>')`** — checks group membership. Returns True if user is *in or implies* that group.
- **`message_post(body=...)`** — adds a chatter entry. Audit trail bonus.
- The empty-report guard (`if not report.observation_ids and outlets_visited == 0`) is a business rule, not framework — adjust to your needs.

Try as a collector: click **Submit** on an empty report → "Add at least one observation..." dialog blocks it. As a collector, try **Approve** → "Only supervisors..." blocks it. As a supervisor → approval works.

## Step 9: the `field_report_count` smart button on res.users

Recall Part 1 added `field_report_count` to `res.users`. Now display it as a smart button on the user form. Add a view inheritance in a new file `views/res_users_views.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <record id="view_users_form_inherit_field_reports" model="ir.ui.view">
        <field name="name">res.users.form.inherit.field_reports</field>
        <field name="model">res.users</field>
        <field name="inherit_id" ref="base.view_users_form"/>
        <field name="arch" type="xml">
            <xpath expr="//div[@name='button_box']" position="inside">
                <button name="action_view_field_reports"
                        type="object"
                        class="oe_stat_button"
                        icon="fa-clipboard">
                    <field name="field_report_count" widget="statinfo" string="Field Reports"/>
                </button>
            </xpath>
        </field>
    </record>

</odoo>
```

Then add the method on the inherited `res.users` in `models/res_users.py`:

```python
def action_view_field_reports(self):
    self.ensure_one()
    return {
        'type': 'ir.actions.act_window',
        'name': f"Field Reports — {self.name}",
        'res_model': 'hcpi.field.report',
        'view_mode': 'list,form,kanban',
        'domain': [('enumerator_id', '=', self.id)],
        'context': {'default_enumerator_id': self.id},
    }
```

Wire the new view file in `__manifest__.py`:

```python
'data': [
    ...
    'views/res_users_views.xml',     # ← new
    ...
],
```

Walk through the inheritance:

- **`inherit_id` ref="base.view_users_form"`** — name of the view we're modifying. `base.view_users_form` is the built-in user form.
- **`<xpath expr="//div[@name='button_box']" position="inside">`** — an XPath into the parent view, plus a position telling Odoo *where* relative to the match.
- **Common positions:** `inside` (add as last child), `before`, `after`, `replace`, `attributes` (modify attrs on the matched node).

After upgrade, open **Settings → Users → admin** → a new **Field Reports** smart button counts how many reports they've filed; clicking it opens the filtered list.

This pattern — `xpath inherit_id` + `position` — is **how country-specific modules add fields to HCPI base models**. See [Country Variants](../../understanding-the-codebase/country-variants.md).

## Step 10: a final check

Stop Odoo, run a full upgrade:

```bash
python odoo/odoo-bin -c conf/hcpi.conf -u hcpi_field_reports
```

Tour:

1. **Settings → Users**: confirm the **Field Reports** category appears with three radio options.
2. Create / log in as `collector1` with Collector only.
3. As `collector1`:
    - Create a report. Note the auto-name (`FR/2026/0001`).
    - Try to delete it — **no delete option** in the Actions menu (ACL forbids).
    - Try to delete an observation — **fine** (Collector has full CRUD on observations).
    - Try to view another user's reports (you won't see them in the list because of the record rule).
4. As admin/manager:
    - Open Settings → Technical → **Database Structure → Record Rules** (or **Security → Record Rules** with debug mode). Filter by Field Report. Confirm both rules exist.
    - View the collector's report. Click **Approve** — `action_approve` fires, state moves, chatter logs "Report approved."
    - Change `outlets_visited`. The chatter records the change (because the field is `tracking=True`).
5. Print the PDF — observations badge-coloured, header + footer rendered.

## Exercises

1. **Add a `reviewer_id` field** to `hcpi.field.report` — a `Many2one('res.users')`, set automatically in `action_approve` to the user who approved. Add a `tracking=True` so changes log. Display on the form.

2. **Constraint with cross-record check**: `_sql_constraints` enforces "one report per enumerator per date." Write the equivalent as `@api.constrains` instead — search for another record with the same enumerator+date and raise if found. (This catches edits that change the date onto an existing pair, which a `unique` constraint also catches at the DB layer — useful as a redundant safety net with a nicer message.)

3. **A scheduled action** (cron job): create an `ir.cron` record that runs every Sunday at 18:00 and posts a chatter message on every approved report from that week noting "Weekly summary." Hint: `ir.cron` records live in XML data files; their `model_id` references a model whose method runs.

4. **A custom group for read-only auditors**: define `group_field_reports_auditor` that doesn't imply collector. Grant read on all models, no write. Confirm a user in only that group can browse everything but cannot create.

5. **A second record rule**: prevent collectors from editing their own report once it's in `submitted` or `approved` state — they can still read it, but writes are blocked. Hint: a write-only rule with `perm_write=True` and a domain restricting to `state == 'draft'`.

## What you learned

After Part 3 you understand the entire surface area of an HCPI module:

- **Groups, ACLs, record rules**, and how they combine.
- **`implied_ids`** for role hierarchies.
- **`ir.module.category`** for clean user-permissions UI.
- **`ir.rule.domain_force`** with `user.id` for "own records only" rules.
- **Sequences** with `next_by_code` and a `create()` override.
- **`@api.model_create_multi`** — the modern batch create signature.
- **`@api.constrains`** vs `_sql_constraints` vs `@api.onchange`.
- **`UserError` vs `ValidationError`** and when to raise each.
- **`mail.thread`** mixin for chatter + tracking; `mail.activity.mixin` for activities.
- **`message_post`** to write to the chatter from Python.
- **`has_group`** for in-method permission checks.
- **View inheritance** with `xpath` + `position` (the pattern country modules use everywhere).

## What now?

The module you built is a real module — clone it, rename it, swap the domain, and you have a working starter for almost any "an enumerator does X periodically, a supervisor reviews" workflow.

From here:

- **[Module Reference](../../understanding-the-codebase/module-reference.md)** — open the real HCPI modules. You'll recognise every piece: the model files, the security files, the inheritance, the chatter, the kanbans, the reports.
- **[Country Variants](../../understanding-the-codebase/country-variants.md)** — see how Uganda, Kenya, Tanzania extend the same base modules with `_inherit` and `xpath` view inheritance.
- **Days 8 onwards** (in the original programme) walk through the actual HCPI modules — Coicop, Location, Outlet, Item, Data Collection, Data Validation, Mobile API. Everything that runs HCPI is the patterns from these three pages, applied to the price-index domain.

## Cleaning up

To remove the practice module entirely when you're done:

1. **Apps → HCPI Field Reports → Uninstall** (drops all tables and data).
2. Stop Odoo, then `rm -rf /opt/hcpi/custom/HCPI/hcpi_field_reports/`.
3. Restart Odoo.

Same pattern as [Your First Module](../../first-edits/your-first-module.md#removing-the-module).
