# Building an HCPI Module — Part 3: Security & Polish

<!--
Duration: half a day (about 4 hours).
Covers: user groups, model-level access (ir.model.access.csv), record-level
rules (ir.rule), sequences for auto-numbering, @api.constrains,
mail.thread chatter and audit trail, button workflows with validation,
the module's final polish.
Prereq: Parts 1 and 2 complete.
-->

You have a working module with data and good UX. Part 3 makes it production-ready: only the right people can see and change the right things, every change is auditable, and the workflow refuses to let you skip steps.

## What "security" means in Odoo

Two distinct layers, each ultimately stored as rows in built-in tables (just like views, actions, and menus — recall the picture from [Odoo Basics](../../understanding-the-codebase/odoo-basics.md)):

| Layer | Stored in | What it controls |
|---|---|---|
| **Groups** | `res.groups` | Who is in which "role." Users belong to groups. |
| **Model access (ACL)** | `ir.model.access` | Per-group, per-model: can the group **C**reate / **R**ead / **U**pdate / **D**elete this model at all? |
| **Record rules** | `ir.rule` | Per-group, per-model: which **rows** can the group see/edit? Implemented as a domain auto-appended to every query. |

The mental model: **ACL is the door, record rules are the floor plan.** ACL says "you may enter the room." Record rules say "and within that room you can only see your own desk."

## Step 1: define the security groups

Create `security/hcpi_outlet_onboarding_security.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <!-- Category — a header in the user permissions UI -->
    <record id="module_category_hcpi_outlet_onboarding" model="ir.module.category">
        <field name="name">HCPI Outlet Onboarding</field>
        <field name="description">Propose, inspect, and approve candidate outlets.</field>
        <field name="sequence">25</field>
    </record>

    <!-- Groups, ordered from least to most privileged -->
    <record id="group_outlet_onboarding_officer" model="res.groups">
        <field name="name">Field Officer</field>
        <field name="category_id" ref="module_category_hcpi_outlet_onboarding"/>
        <field name="comment">File proposals, perform inspections on their own proposals.</field>
    </record>

    <record id="group_outlet_onboarding_supervisor" model="res.groups">
        <field name="name">Supervisor</field>
        <field name="category_id" ref="module_category_hcpi_outlet_onboarding"/>
        <field name="implied_ids" eval="[(4, ref('group_outlet_onboarding_officer'))]"/>
        <field name="comment">Review proposals from any officer; perform inspections anywhere.</field>
    </record>

    <record id="group_outlet_onboarding_manager" model="res.groups">
        <field name="name">Manager</field>
        <field name="category_id" ref="module_category_hcpi_outlet_onboarding"/>
        <field name="implied_ids" eval="[(4, ref('group_outlet_onboarding_supervisor'))]"/>
        <field name="comment">Approve or reject proposals; manage configuration.</field>
    </record>

</odoo>
```

Walk through:

- **`ir.module.category`** — a UI grouping in **Settings → Users → User → Access Rights**. Without a category, your three groups show up as a dropdown rather than radio buttons grouped under "HCPI Outlet Onboarding."
- **`res.groups`** — the actual group records.
- **`implied_ids = [(4, ref('group_outlet_onboarding_officer'))]`** — "anyone in Supervisor is automatically also in Field Officer." `(4, ref('...'))` is the link-an-existing-record command tuple you learned in [Part 1](part1-models.md#command-tuples-for-relational-writes).
- The triangle relation **Manager → Supervisor → Field Officer** means a Manager has all the rights of the levels below. This is the standard Odoo pattern.

Add the file to `__manifest__.py` `data` (security files load **before** views — security records must exist before views reference them):

```python
'data': [
    'security/hcpi_outlet_onboarding_security.xml',     # ← new, FIRST
    'security/ir.model.access.csv',
    'reports/hcpi_outlet_proposal_report.xml',
    'views/hcpi_region_views.xml',
    ...
],
```

## Step 2: replace the open ACL with a proper matrix

Open `security/ir.model.access.csv` and replace its content:

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink

access_hcpi_region_officer,hcpi.region officer read,model_hcpi_region,hcpi_outlet_onboarding.group_outlet_onboarding_officer,1,0,0,0
access_hcpi_region_manager,hcpi.region manager all,model_hcpi_region,hcpi_outlet_onboarding.group_outlet_onboarding_manager,1,1,1,1

access_hcpi_outlet_type_officer,hcpi.outlet.type officer read,model_hcpi_outlet_type,hcpi_outlet_onboarding.group_outlet_onboarding_officer,1,0,0,0
access_hcpi_outlet_type_manager,hcpi.outlet.type manager all,model_hcpi_outlet_type,hcpi_outlet_onboarding.group_outlet_onboarding_manager,1,1,1,1

access_hcpi_outlet_tag_officer,hcpi.outlet.tag officer read,model_hcpi_outlet_tag,hcpi_outlet_onboarding.group_outlet_onboarding_officer,1,0,0,0
access_hcpi_outlet_tag_supervisor,hcpi.outlet.tag supervisor all,model_hcpi_outlet_tag,hcpi_outlet_onboarding.group_outlet_onboarding_supervisor,1,1,1,1

access_hcpi_outlet_inspection_officer,hcpi.outlet.inspection officer all,model_hcpi_outlet_inspection,hcpi_outlet_onboarding.group_outlet_onboarding_officer,1,1,1,1
access_hcpi_outlet_inspection_supervisor,hcpi.outlet.inspection supervisor all,model_hcpi_outlet_inspection,hcpi_outlet_onboarding.group_outlet_onboarding_supervisor,1,1,1,1

access_hcpi_outlet_proposal_officer,hcpi.outlet.proposal officer,model_hcpi_outlet_proposal,hcpi_outlet_onboarding.group_outlet_onboarding_officer,1,1,1,0
access_hcpi_outlet_proposal_supervisor,hcpi.outlet.proposal supervisor,model_hcpi_outlet_proposal,hcpi_outlet_onboarding.group_outlet_onboarding_supervisor,1,1,1,0
access_hcpi_outlet_proposal_manager,hcpi.outlet.proposal manager,model_hcpi_outlet_proposal,hcpi_outlet_onboarding.group_outlet_onboarding_manager,1,1,1,1
```

Reading row by row:

| Model | Group | C | R | U | D | Why |
|---|---|---|---|---|---|---|
| `hcpi.region` | Field Officer | 0 | 1 | 0 | 0 | Need to *pick* a region but shouldn't edit master data. |
| `hcpi.region` | Manager | 1 | 1 | 1 | 1 | Master data administration. |
| `hcpi.outlet.type` | Field Officer | 0 | 1 | 0 | 0 | Pick types, don't create new ones ad-hoc. |
| `hcpi.outlet.type` | Manager | 1 | 1 | 1 | 1 | Manage type catalogue. |
| `hcpi.outlet.tag` | Field Officer | 0 | 1 | 0 | 0 | Pick existing tags. |
| `hcpi.outlet.tag` | Supervisor | 1 | 1 | 1 | 1 | Supervisors can mint new tags. |
| `hcpi.outlet.inspection` | Field Officer | 1 | 1 | 1 | 1 | They author inspections on proposals they can see. |
| `hcpi.outlet.proposal` | Field Officer | 1 | 1 | 1 | 0 | Create and edit, but **no delete** — keep an audit trail. |
| `hcpi.outlet.proposal` | Manager | 1 | 1 | 1 | 1 | Manager can delete (e.g., garbage cleanup). |

**Key principle:** ACL is binary at the model level. Use ACL for the rough "can this role even touch this model?" question. For nuance (this row but not that row) use **record rules** in the next step.

## Step 3: record rules — "you only see your own proposals"

A field officer should see only proposals they filed; a supervisor sees everyone's. We codify that with `ir.rule`.

Add to `security/hcpi_outlet_onboarding_security.xml`, after the groups:

```xml
<!-- Field officer: only their OWN proposals -->
<record id="rule_proposal_own" model="ir.rule">
    <field name="name">Outlet Proposal: own only</field>
    <field name="model_id" ref="model_hcpi_outlet_proposal"/>
    <field name="domain_force">[('proposed_by', '=', user.id)]</field>
    <field name="groups" eval="[(4, ref('group_outlet_onboarding_officer'))]"/>
    <field name="perm_read" eval="True"/>
    <field name="perm_write" eval="True"/>
    <field name="perm_create" eval="True"/>
    <field name="perm_unlink" eval="True"/>
</record>

<!-- Supervisor: sees ALL proposals (override the more restrictive rule above) -->
<record id="rule_proposal_supervisor_all" model="ir.rule">
    <field name="name">Outlet Proposal: supervisor sees all</field>
    <field name="model_id" ref="model_hcpi_outlet_proposal"/>
    <field name="domain_force">[(1, '=', 1)]</field>
    <field name="groups" eval="[(4, ref('group_outlet_onboarding_supervisor'))]"/>
</record>

<!-- Field officer: only inspections on their own proposals -->
<record id="rule_inspection_own" model="ir.rule">
    <field name="name">Outlet Inspection: own proposals only</field>
    <field name="model_id" ref="model_hcpi_outlet_inspection"/>
    <field name="domain_force">[('proposal_id.proposed_by', '=', user.id)]</field>
    <field name="groups" eval="[(4, ref('group_outlet_onboarding_officer'))]"/>
</record>

<record id="rule_inspection_supervisor_all" model="ir.rule">
    <field name="name">Outlet Inspection: supervisor sees all</field>
    <field name="model_id" ref="model_hcpi_outlet_inspection"/>
    <field name="domain_force">[(1, '=', 1)]</field>
    <field name="groups" eval="[(4, ref('group_outlet_onboarding_supervisor'))]"/>
</record>
```

Reading this:

- **`domain_force`** — a domain auto-appended to every search. The user never sees rows that don't match.
- **`user.id`** — magic variable in record rules: the currently logged-in user's id.
- **`(1, '=', 1)`** — the "match everything" domain. Used to give a higher-privilege group full visibility.
- **`perm_read` / `perm_write` / `perm_create` / `perm_unlink`** — which operations the rule applies to. If omitted, the rule applies to **all four**. We're explicit on the first rule for clarity.

### How rules combine

This trips everyone up at first:

**Across groups: rules are OR'd.** If a user is in both Field Officer and Supervisor, they see the union of what each rule allows. That's why the supervisor rule with `(1, '=', 1)` effectively grants full visibility.

**Within one group, rules are AND'd.** If a single group has two rules, both must match.

So for our triangle (Supervisor implies Field Officer), a supervisor user has two applicable rules. The Supervisor's `(1, '=', 1)` OR'd with the Officer's `proposed_by = user.id` simplifies to `True`. They see everything.

### Global rules vs group-specific rules

A rule with `global=True` (no groups) applies to **everyone** and is AND'd with group rules. We don't need global rules here — the rule applies only to the listed groups via `groups`.

## Step 4: install groups onto the admin

Restart Odoo with the upgrade flag:

```bash
python odoo/odoo-bin -c conf/hcpi.conf -u hcpi_outlet_onboarding
```

In the UI, **Settings → Users & Companies → Users → Administrator → Access Rights** tab. Under "HCPI Outlet Onboarding" you should see your three radio buttons:

- Field Officer
- Supervisor
- Manager (selected automatically — admin gets everything)

If you want to test the rules properly:

1. Create a second user **officer1** with email + name. On their **Access Rights** tab, give them "HCPI Outlet Onboarding → Field Officer" (and nothing else).
2. Set their password (top of user form → ⋮ → **Change Password**).
3. Open a private/incognito browser window and log in as `officer1`.
4. Open **Outlet Onboarding → Proposals** → you see **only your own proposals** (which is none yet, until you create one as `officer1`).

File one proposal as `officer1`. Then log back in as admin — you see both your own admin proposal and the `officer1` one. Rules working.

## Step 5: a sequence for auto-numbering

Replace `name="New"` with proper references like `OP/2026/0001`.

Create `data/hcpi_outlet_proposal_data.xml`:

```bash
mkdir -p /opt/hcpi/custom/HCPI/hcpi_outlet_onboarding/data
touch /opt/hcpi/custom/HCPI/hcpi_outlet_onboarding/data/hcpi_outlet_proposal_data.xml
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="seq_hcpi_outlet_proposal" model="ir.sequence">
        <field name="name">HCPI Outlet Proposal</field>
        <field name="code">hcpi.outlet.proposal</field>
        <field name="prefix">OP/%(year)s/</field>
        <field name="padding">4</field>
        <field name="company_id" eval="False"/>
    </record>
</odoo>
```

- **`code`** — the handle used in Python (`env['ir.sequence'].next_by_code('hcpi.outlet.proposal')`).
- **`prefix`** — `%(year)s` is auto-replaced with the current year. Other tokens: `%(month)s`, `%(day)s`, etc.
- **`padding=4`** — pad the counter to 4 digits with leading zeros: `0001`, `0002`, …
- **`company_id` eval="False"`** — no per-company numbering. (For multi-company instances you might want a different value here.)

Add to `__manifest__.py`:

```python
'data': [
    'security/hcpi_outlet_onboarding_security.xml',
    'security/ir.model.access.csv',
    'data/hcpi_outlet_proposal_data.xml',          # ← new
    'reports/hcpi_outlet_proposal_report.xml',
    'views/hcpi_region_views.xml',
    ...
],
```

Now override `create()` on the proposal model so newly-created proposals pull from the sequence. In `models/hcpi_outlet_proposal.py`, add at the bottom of the class:

```python
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        if vals.get('name', "New") == "New":
            vals['name'] = self.env['ir.sequence'].next_by_code('hcpi.outlet.proposal') or "New"
    return super().create(vals_list)
```

Reading the override:

- **`@api.model_create_multi`** — Odoo 17+ batch-create decorator. `vals_list` is a list of dicts (one per new record). This signature replaces the older single-dict `create()`.
- **The `"New"` guard** — only generate a sequence number if the caller didn't already pass a name (so duplication / import flows keep their names).
- **`super().create(vals_list)`** — chain to the parent (the framework's default behaviour) so the records actually get created.

After upgrade (`-u hcpi_outlet_onboarding`) and creating a new proposal, the reference should read `OP/2026/0001`.

## Step 6: mail.thread — chatter and audit trail

The chatter is that comments/log section at the bottom of forms in Odoo. It's not just chat — it's also where field changes are logged when fields are marked `tracking=True`. You already added `tracking=True` to `state` in Part 1; now we wire up the mixin to make it visible.

In `models/hcpi_outlet_proposal.py`, change the class header:

```python
class HcpiOutletProposal(models.Model):
    _name = 'hcpi.outlet.proposal'
    _description = "Outlet Proposal"
    _inherit = ['mail.thread', 'mail.activity.mixin']
    _order = 'create_date desc'
```

That's all the Python side needs. `mail.thread` adds:

- A `message_ids` One2many to all chatter messages.
- A `message_follower_ids` for follower management.
- Auto-tracking of fields with `tracking=True`.

`mail.activity.mixin` adds:

- `activity_ids` for scheduled activities ("call this contact next Tuesday").

Then add the chatter to the form view. At the bottom of the form, after closing `</sheet>` but inside `</form>`:

```xml
        </sheet>
        <chatter/>
    </form>
```

The `<chatter/>` tag is Odoo 17/18 shorthand. Older code uses an expanded `<div class="oe_chatter">` block — same effect.

Upgrade. Open a proposal. At the bottom of the form a chatter panel now appears:

- **Send message** — adds a comment.
- **Log note** — internal-only note.
- **Followers** — manage who gets notified.
- Above the input, the **log feed** — and you'll see the `state` change you made earlier already recorded as **State: Draft → Inspecting**.

Make a few state changes via the workflow buttons. Each one gets logged automatically.

### Tracking more fields

Pick any field where you want history. Add `tracking=True`:

```python
outlet_name = fields.Char(string="Outlet Name", required=True, tracking=True)
region_id = fields.Many2one('hcpi.region', string="Region", required=True, index=True, tracking=True)
outlet_type_id = fields.Many2one('hcpi.outlet.type', string="Outlet Type", required=True, tracking=True)
contact_phone = fields.Char(tracking=True)
```

Upgrade — every change to these fields shows up in the chatter as `Field: old → new`.

## Step 7: validation with `@api.constrains`

Right now nothing prevents an officer from submitting a proposal with garbage GPS coordinates. Add Python constraints.

In `models/hcpi_outlet_proposal.py`, add the import at the top:

```python
from odoo.exceptions import UserError, ValidationError
```

Then inside the class:

```python
@api.constrains('latitude', 'longitude')
def _check_gps_range(self):
    for proposal in self:
        if proposal.latitude and not (-90 <= proposal.latitude <= 90):
            raise ValidationError("Latitude must be between -90 and 90.")
        if proposal.longitude and not (-180 <= proposal.longitude <= 180):
            raise ValidationError("Longitude must be between -180 and 180.")

@api.constrains('contact_phone')
def _check_phone_length(self):
    for proposal in self:
        if proposal.contact_phone and len(proposal.contact_phone.strip()) < 7:
            raise ValidationError("Contact phone looks too short to be valid.")
```

`@api.constrains` runs on every save. Different from `@api.onchange` (which runs in the UI as the user types but doesn't block save — it's for hints) and from `_sql_constraints` (database-level, faster but limited to what SQL can express).

Try saving a proposal with `latitude = 999` → red error toast appears, save blocked.

## Step 8: harden the workflow buttons

Right now `action_start_inspection` lets anyone advance any draft. Add some sanity checks plus role-based gating. Replace the action methods in `models/hcpi_outlet_proposal.py`:

```python
def action_start_inspection(self):
    for proposal in self:
        if proposal.state != 'draft':
            raise UserError("Only draft proposals can move to Inspecting.")
        if not proposal.region_id or not proposal.outlet_type_id:
            raise UserError("Set the region and outlet type before starting inspection.")
        proposal.state = 'inspecting'
        proposal.message_post(body="Moved to inspection.")

def action_submit_for_review(self):
    for proposal in self:
        if proposal.state != 'inspecting':
            raise UserError("Only proposals under inspection can be submitted for review.")
        if not proposal.inspection_ids:
            raise UserError("Record at least one inspection before submitting for review.")
        if proposal.has_failed_inspection:
            raise UserError("Cannot submit a proposal with a failed inspection. Reset or re-inspect first.")
        proposal.state = 'review'
        proposal.message_post(body="Submitted for management review.")

def action_approve(self):
    self._require_manager()
    for proposal in self:
        if proposal.state != 'review':
            raise UserError("Only proposals under review can be approved.")
        proposal.state = 'approved'
        proposal.reviewed_by = self.env.user
        proposal.approval_date = fields.Date.context_today(proposal)
        proposal.message_post(body="Proposal approved.")

def action_reject(self):
    self._require_supervisor()
    for proposal in self:
        if proposal.state not in ('review', 'inspecting'):
            raise UserError("Only proposals under inspection or review can be rejected.")
        proposal.state = 'rejected'
        proposal.reviewed_by = self.env.user
        proposal.message_post(body="Proposal rejected.")

def action_reset_to_draft(self):
    self._require_supervisor()
    for proposal in self:
        proposal.state = 'draft'
        proposal.reviewed_by = False
        proposal.approval_date = False
        proposal.message_post(body="Reset to draft.")

def _require_supervisor(self):
    if not self.env.user.has_group('hcpi_outlet_onboarding.group_outlet_onboarding_supervisor'):
        raise UserError("Only supervisors can perform this action.")

def _require_manager(self):
    if not self.env.user.has_group('hcpi_outlet_onboarding.group_outlet_onboarding_manager'):
        raise UserError("Only managers can approve proposals.")
```

Reading the changes:

- **`raise UserError(...)`** — friendly dialog to the user. Different from `ValidationError` mainly by convention; both stop the operation and surface their message.
- **`has_group('<xml-id>')`** — checks group membership. Returns True if the user is *in or implies* that group, so a Manager passes `has_group('...supervisor')`.
- **`message_post(body=...)`** — adds a chatter entry. Audit trail bonus on top of the automatic `tracking=True` logs.
- The business rules (region+type before inspection, at least one inspection before review, no failed inspections at submission) encode the actual workflow constraints — adjust to match your country's intake policy.

Try as a Field Officer: click **Submit for Review** on a proposal with no inspections → "Record at least one inspection..." dialog blocks it. As a Field Officer, try **Approve** → "Only managers..." blocks it. As a Manager → approval works.

## Step 9: the `proposal_count` smart button on res.users

Recall Part 1 added `proposal_count` to `res.users`. Now display it as a smart button on the user form. Create `views/res_users_views.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <record id="view_users_form_inherit_outlet_onboarding" model="ir.ui.view">
        <field name="name">res.users.form.inherit.outlet_onboarding</field>
        <field name="model">res.users</field>
        <field name="inherit_id" ref="base.view_users_form"/>
        <field name="arch" type="xml">
            <xpath expr="//div[@name='button_box']" position="inside">
                <button name="action_view_proposals"
                        type="object"
                        class="oe_stat_button"
                        icon="fa-building">
                    <field name="proposal_count" widget="statinfo" string="Proposals"/>
                </button>
            </xpath>
        </field>
    </record>

</odoo>
```

Add the method on the inherited `res.users` in `models/res_users.py`:

```python
def action_view_proposals(self):
    self.ensure_one()
    return {
        'type': 'ir.actions.act_window',
        'name': f"Outlet Proposals — {self.name}",
        'res_model': 'hcpi.outlet.proposal',
        'view_mode': 'list,form,kanban',
        'domain': [('proposed_by', '=', self.id)],
        'context': {'default_proposed_by': self.id},
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

After upgrade, open **Settings → Users → admin** → a new **Proposals** smart button counts how many proposals they've filed; clicking it opens the filtered list.

This pattern — `xpath inherit_id` + `position` — is **how country-specific modules add fields to HCPI base models**. See [Country Variants](../../understanding-the-codebase/country-variants.md).

## Step 10: a final check

Stop Odoo, run a full upgrade:

```bash
python odoo/odoo-bin -c conf/hcpi.conf -u hcpi_outlet_onboarding
```

Tour:

1. **Settings → Users**: confirm the **HCPI Outlet Onboarding** category appears with three radio options.
2. Create / log in as `officer1` with Field Officer only.
3. As `officer1`:
    - Create a proposal. Note the auto-name (`OP/2026/0001`).
    - Try to delete it — **no delete option** in the Actions menu (ACL forbids).
    - Try to delete an inspection — **fine** (Field Officer has full CRUD on inspections through the record rule).
    - Try to view another user's proposals (you won't see them in the list because of the record rule).
4. As admin/manager:
    - Open Settings → Technical → **Database Structure → Record Rules** (or **Security → Record Rules** with debug mode). Filter by Outlet Proposal. Confirm both rules exist.
    - View the officer's proposal. Click **Approve** — `action_approve` fires, state moves, chatter logs "Proposal approved."
    - Change `contact_phone`. The chatter records the change (because the field is `tracking=True`).
5. Print the PDF dossier — inspection results badge-coloured, header + footer rendered.

## Exercises

1. **Add an `approved_outlet_count` field on `hcpi.region`** — a stored compute counting how many approved proposals are in that region (and any sub-regions). Hint: `child_of` domain in a search inside the compute.

2. **Constraint with cross-record check**: `_sql_constraints` enforces "one proposal per outlet name per region." Write the equivalent as `@api.constrains` instead — search for another record with the same outlet_name+region and raise if found. Useful as a redundant safety net with a nicer message than PostgreSQL's default.

3. **A scheduled action** (cron job): create an `ir.cron` record that runs every Monday at 09:00 and posts a chatter message on every proposal that's been in `review` for more than 7 days, nudging the manager. Hint: `ir.cron` records live in data XML; their `model_id` references the model whose method runs.

4. **A read-only auditor group**: define `group_outlet_onboarding_auditor` that doesn't imply field officer. Grant read on all models, no write. Confirm a user in only that group can browse everything but cannot create.

5. **A second record rule**: prevent field officers from editing their own proposal once it's in `review` or `approved` state — they can still read it, but writes are blocked. Hint: a write-only rule with `perm_write=True` and a domain restricting to `state in ('draft', 'inspecting')`.

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

The module you built is a real module — clone it, rename it, swap the domain, and you have a working starter for almost any "an officer proposes, a reviewer inspects, a manager approves" workflow.

From here:

- **[Module Reference](../../understanding-the-codebase/module-reference.md)** — open the real HCPI modules. You'll recognise every piece: the model files, the security files, the inheritance, the chatter, the kanbans, the reports.
- **[Country Variants](../../understanding-the-codebase/country-variants.md)** — see how Uganda, Kenya, Tanzania extend the same base modules with `_inherit` and `xpath` view inheritance.
- The remaining training topics (Coicop, Location, Outlets, Items, Data Collection, Validation, Mobile API) walk through the actual HCPI modules — everything that runs HCPI is the patterns from these three pages, applied to the price-index domain.

## Cleaning up

To remove the practice module entirely when you're done:

1. **Apps → HCPI Outlet Onboarding → Uninstall** (drops all tables and data).
2. Stop Odoo, then `rm -rf /opt/hcpi/custom/HCPI/hcpi_outlet_onboarding/`.
3. Restart Odoo.

Uninstall before deleting the folder — otherwise Odoo will warn you on every restart about the missing module that still has database records.
