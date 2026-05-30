# Building an HCPI Module — Part 3: Rights, Polish & Controllers

<!--
Duration: about 1 hour.
Covers: HCPI's security conventions, groups, ACL matrix, record rules,
GPS validation with @api.constrains, sequence-based auto-numbering,
mail.thread chatter, hardened workflow buttons, and a closing section
on HTTP controllers — what they are, why this module has no
controllers/ folder, and what RPC endpoint replaces them.
-->

You have a working module with data and good UX. Part 3 closes it out: who can do what, a handful of polish features every real HCPI module has, and a short note on the one piece this module deliberately doesn't have — controllers.

## Two security layers

Two distinct layers, each ultimately stored as rows in built-in tables (just like views, actions, and menus — recall the picture from [Odoo Basics](../../understanding-the-codebase/odoo-basics.md)):

| Layer | Stored in | What it controls |
|---|---|---|
| **Groups** | `res.groups` | Who is in which "role." Users belong to groups. |
| **Model access (ACL)** | `ir.model.access` | Per-group, per-model: can the group **C**reate / **R**ead / **U**pdate / **D**elete this model at all? |
| **Record rules** | `ir.rule` | Per-group, per-model: which **rows** can the group see/edit? Implemented as a domain auto-appended to every query. |

The mental model: **ACL is the door, record rules are the floor plan.** ACL says "you may enter the room." Record rules say "and within that room you can only see your own desk."

## How real HCPI modules set this up

Before we wire up our own groups, look at how the production modules do it. The pattern is consistent across HCPI and worth recognising:

| Module | Groups defined | Pattern |
|---|---|---|
| **`hcpi_coicop`** | `coicop_user` (read-only), `coicop_manager` (full CRUD) | The simplest pattern — one read group, one admin group. Used by all the reference-data modules (`hcpi_item`, `hcpi_brand`, locations). |
| **`hcpi_outlet`** | `group_outlet_user`, `group_outlet_manager` | Same `_user` / `_manager` shape. Outlets are master data; reading is broad, editing is restricted. |
| **`hcpi_data_collection`** | `group_data_collection_collector`, `group_data_collection_supervisor`, `group_data_collection_statician` | Three roles for an actual workflow. Each role implies the one below: a statistician implies supervisor implies collector. |
| **`hcpi_index`** | `group_index_viewer`, `group_index_manager` | Reading the published indices is broad; computing/republishing is restricted. |

Two takeaways:

1. **The default pattern is a `_user` / `_manager` pair.** If a module's only job is "show this reference data; let admins edit it," that's the whole security model.
2. **Workflow modules use a role triangle** with `implied_ids` so the higher roles inherit everything below them. `hcpi_data_collection` is the canonical example — a collector can submit collections, a supervisor can also approve them, a statistician can also re-run validation algorithms.

Our module fits between the two. We'll add **one** custom group — `Manager` — and contrast it with regular internal users (`base.group_user`). That's enough to demonstrate every primitive without the triangle complexity. Real HCPI modules that need roles add the triangle.

## Step 1: define the Manager group

Create `security/hcpi_outlet_onboarding_security.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <!-- Category — a header in the user permissions UI -->
    <record id="module_category_hcpi_outlet_onboarding" model="ir.module.category">
        <field name="name">HCPI Outlet Onboarding</field>
        <field name="sequence">25</field>
    </record>

    <!-- One custom group: Manager. Every other internal user is a regular user. -->
    <record id="group_outlet_onboarding_manager" model="res.groups">
        <field name="name">Manager</field>
        <field name="category_id" ref="module_category_hcpi_outlet_onboarding"/>
        <field name="comment">Approve proposals and see everyone's work.</field>
    </record>

</odoo>
```

- **`ir.module.category`** — a UI grouping in **Settings → Users → User → Access Rights**. Without it, your group shows up as a flat checkbox rather than under "HCPI Outlet Onboarding."
- **`res.groups`** — the actual group record.

Add the file to `__manifest__.py` `data` (security files load **before** views — security records must exist before views reference them):

```python
'data': [
    'security/hcpi_outlet_onboarding_security.xml',     # ← new, FIRST
    'security/ir.model.access.csv',
    'views/views.xml',
],
```

## Step 2: refine the ACL matrix

Open `security/ir.model.access.csv` and replace its content:

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_hcpi_outlet_visit_user,hcpi.outlet.visit user,model_hcpi_outlet_visit,base.group_user,1,1,1,1
access_hcpi_outlet_proposal_user,hcpi.outlet.proposal user,model_hcpi_outlet_proposal,base.group_user,1,1,1,0
access_hcpi_outlet_proposal_manager,hcpi.outlet.proposal manager,model_hcpi_outlet_proposal,hcpi_outlet_onboarding.group_outlet_onboarding_manager,1,1,1,1
```

Reading row by row:

| Model | Group | C | R | U | D | Why |
|---|---|---|---|---|---|---|
| `hcpi.outlet.visit` | Every user | 1 | 1 | 1 | 1 | Anyone can log a visit. |
| `hcpi.outlet.proposal` | Every user | 1 | 1 | 1 | 0 | Create and edit, but **no delete** — keep an audit trail. |
| `hcpi.outlet.proposal` | Manager | 1 | 1 | 1 | 1 | Manager can also delete (garbage cleanup). |

**Key principle:** ACL is binary at the model level. Use it for the rough "can this role even touch this model?" question. For nuance (this row but not that row) use **record rules** in the next step.

**How ACLs combine across groups:** they're OR'd. A user in both `base.group_user` AND `Manager` has the union of their permissions — so a Manager can delete proposals because *that* group has unlink, even though the regular-user row doesn't.

## Step 3: a record rule — "you only see your own proposals"

A regular user should see only proposals they filed; a Manager sees everyone's. We codify that with `ir.rule`.

Add to `security/hcpi_outlet_onboarding_security.xml`, after the group:

```xml
<!-- Regular users: only their OWN proposals -->
<record id="rule_proposal_own" model="ir.rule">
    <field name="name">Outlet Proposal: own only</field>
    <field name="model_id" ref="model_hcpi_outlet_proposal"/>
    <field name="domain_force">[('proposed_by', '=', user.id)]</field>
    <field name="groups" eval="[(4, ref('base.group_user'))]"/>
</record>

<!-- Managers: see ALL proposals -->
<record id="rule_proposal_manager_all" model="ir.rule">
    <field name="name">Outlet Proposal: manager sees all</field>
    <field name="model_id" ref="model_hcpi_outlet_proposal"/>
    <field name="domain_force">[(1, '=', 1)]</field>
    <field name="groups" eval="[(4, ref('group_outlet_onboarding_manager'))]"/>
</record>
```

Reading this:

- **`domain_force`** — a domain auto-appended to every search. The user never sees rows that don't match.
- **`user.id`** — magic variable in record rules: the currently logged-in user's id.
- **`(1, '=', 1)`** — the "match everything" domain.
- **`(4, ref('base.group_user'))`** — the link-an-existing-record command tuple from [Part 1](part1-models.md). Same shape as `assigned_user_ids` updates.

### How rules combine

**Across groups: rules are OR'd.** A Manager is also in `base.group_user`, so they have two applicable rules. `True OR (proposed_by = user.id)` simplifies to `True` — they see everything.

**Within one group, rules are AND'd.** If a single group has two rules, both must match.

### Verify

Restart Odoo with the upgrade flag:

```bash
python odoo/odoo-bin -c conf/hcpi.conf -u hcpi_outlet_onboarding
```

In **Settings → Users**, the **HCPI Outlet Onboarding** category appears with a **Manager** checkbox. Admin has it by default. Create a second user with no Manager tick, log in as them in a private window, and confirm they only see their own proposals. Try deleting a proposal as them — no Delete option in the Actions menu (ACL forbids).

## Step 4: GPS validation with `@api.constrains`

Right now nothing prevents an officer from saving a proposal with garbage coordinates like `latitude = 999`. Add a Python constraint.

Open `models/hcpi_outlet_proposal.py`. Add the import at the top of the file:

```python
from odoo.exceptions import UserError, ValidationError
```

Inside the class:

```python
@api.constrains('latitude', 'longitude')
def _check_gps_range(self):
    for proposal in self:
        if proposal.latitude and not (-90 <= proposal.latitude <= 90):
            raise ValidationError("Latitude must be between -90 and 90.")
        if proposal.longitude and not (-180 <= proposal.longitude <= 180):
            raise ValidationError("Longitude must be between -180 and 180.")

@api.constrains('contact_phone')
def _check_phone(self):
    for proposal in self:
        if proposal.contact_phone and len(proposal.contact_phone.strip()) < 7:
            raise ValidationError("Contact phone looks too short to be valid.")
```

`@api.constrains` runs on every save. The differences from the alternatives:

| Approach | Where it runs | When it fires | Use for |
|---|---|---|---|
| `_sql_constraints` | PostgreSQL | INSERT/UPDATE | Uniqueness, simple `CHECK` clauses |
| `@api.constrains` | Python, on save | After write | Anything more complex; cross-record checks |
| `@api.onchange` | Python, in the UI | As the user types | Soft hints — *doesn't* block save |

Try saving a proposal with `latitude = 999` → red error toast appears, save blocked.

## Step 5: a sequence for auto-numbering

Replace the placeholder `name="New"` from Part 1 with proper references like `OP/2026/0001`.

Create `data/hcpi_outlet_proposal_data.xml`:

```bash
mkdir -p /opt/hcpi/custom/HCPI/hcpi_outlet_onboarding/data
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
- **`prefix`** — `%(year)s` is auto-replaced with the current year. Other tokens: `%(month)s`, `%(day)s`.
- **`padding=4`** — pad the counter to 4 digits: `0001`, `0002`, …

Add to `__manifest__.py`:

```python
'data': [
    'security/hcpi_outlet_onboarding_security.xml',
    'security/ir.model.access.csv',
    'data/hcpi_outlet_proposal_data.xml',          # ← new
    'views/views.xml',
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

- **`@api.model_create_multi`** — the modern batch-create decorator. `vals_list` is a list of dicts (one per new record). This signature replaces the older single-dict `create()`.
- **The `"New"` guard** — only generate a sequence number if the caller didn't already pass a name (so duplication / import flows keep their names).
- **`super().create(vals_list)`** — chain to the parent so the records actually get created.

After upgrade (`-u hcpi_outlet_onboarding`) and creating a new proposal, the reference reads `OP/2026/0001`.

## Step 6: `mail.thread` chatter and audit trail

The chatter is that comments/log section at the bottom of forms in Odoo. It's not just chat — it's where field changes are logged when fields are marked `tracking=True`. You already added `tracking=True` to `state` in Part 1; now we wire up the mixin to make it visible.

In `models/hcpi_outlet_proposal.py`, change the class header:

```python
class HcpiOutletProposal(models.Model):
    _name = 'hcpi.outlet.proposal'
    _description = "Outlet Proposal"
    _inherit = ['mail.thread', 'mail.activity.mixin']
    _order = 'create_date desc'
```

`_inherit` on a model with its own `_name` is called **prototype inheritance** — you're mixing in features from abstract models. `mail.thread` adds:

- A `message_ids` One2many to all chatter messages.
- A `message_follower_ids` for follower management.
- Auto-tracking of fields with `tracking=True`.

`mail.activity.mixin` adds `activity_ids` for scheduled activities ("call this contact next Tuesday").

Then add the chatter to the form view. At the bottom of the proposal form, after closing `</sheet>` but inside `</form>`:

```xml
        </sheet>
        <chatter/>
    </form>
```

The `<chatter/>` tag is Odoo 17/18 shorthand. Upgrade, open a proposal — a chatter panel now appears at the bottom with **Send message**, **Log note**, **Followers**, and the log feed where every `state` change is recorded as **State: Draft → Active**.

### Tracking more fields

Pick any field where you want history. Add `tracking=True`:

```python
outlet_name = fields.Char(string="Outlet Name", required=True, tracking=True)
outlet_type = fields.Selection([...], required=True, default='open_market', tracking=True)
contact_phone = fields.Char(tracking=True)
```

Every change shows up in the chatter as `Field: old → new`.

## Step 7: harden the workflow buttons

Right now `action_approve` from Part 2 lets anyone advance any draft. Add a state check, a sanity check, and a manager gate. Replace the methods in `models/hcpi_outlet_proposal.py`:

```python
def action_approve(self):
    self._require_manager()
    for proposal in self:
        if proposal.state != 'draft':
            raise UserError("Only draft proposals can be approved.")
        if not proposal.visit_ids:
            raise UserError("Record at least one visit before approving.")
        proposal.state = 'active'
        proposal.message_post(body="Proposal approved.")

def action_retire(self):
    for proposal in self:
        if proposal.state != 'active':
            raise UserError("Only active outlets can be retired.")
        proposal.state = 'retired'
        proposal.message_post(body="Outlet retired.")

def action_reset_to_draft(self):
    for proposal in self:
        proposal.state = 'draft'
        proposal.message_post(body="Reset to draft.")

def _require_manager(self):
    if not self.env.user.has_group('hcpi_outlet_onboarding.group_outlet_onboarding_manager'):
        raise UserError("Only managers can approve proposals.")
```

Reading the changes:

- **`raise UserError(...)`** — friendly dialog to the user. Different from `ValidationError` mainly by convention; both stop the operation and surface their message.
- **`has_group('<xml-id>')`** — checks group membership. Returns True if the user is *in* that group.
- **`message_post(body=...)`** — adds a chatter entry. Audit-trail bonus on top of the automatic `tracking=True` logs.

Try as a regular user: click **Approve** → "Only managers..." blocks it. Try **Approve** with no visits → "Record at least one visit..." blocks it. As a Manager with at least one visit → approval works, and the chatter logs "Proposal approved."

## Step 8: a final tour

Restart with `-u hcpi_outlet_onboarding` one last time, then walk through everything:

1. **Settings → Users**: confirm the **HCPI Outlet Onboarding** category appears with a **Manager** checkbox. Admin has it by default; the regular user you created does not.
2. As the regular user:
    - Create a proposal. Note the auto-name (`OP/2026/0001` from the sequence).
    - Set garbage GPS (`latitude = 999`) and try to save → blocked by `@api.constrains`.
    - Try **Approve** → blocked by `_require_manager`.
    - The list shows only proposals you created — record rule working.
    - Try to delete a proposal → no Delete option in the Actions menu — ACL working.
3. As admin (a Manager by default):
    - View the regular user's proposal. Click **Approve** with a visit recorded — state moves to Active, chatter logs "Proposal approved."
    - Change `contact_phone`. The chatter records the change because the field is `tracking=True`.
    - Delete a proposal you no longer want — manager has unlink.

That's a production-feeling module. Onto the one thing it deliberately *doesn't* have.

## Controllers — and the RPC endpoint that replaces them

Scroll back to [Part 1, Step 1](part1-models.md#step-1-scaffold-the-module). When we scaffolded the module, the generator created a `controllers/` folder. We deleted it before doing anything else. Here's what it would have held, why we don't need it, and what HCPI uses instead.

### What a controller is

An Odoo **controller** is a Python class that registers **HTTP routes** — URLs that Odoo answers when a browser, a mobile app, or any HTTP client hits them. Conceptually identical to a Flask, Django, or Express route handler:

```python
from odoo import http
from odoo.http import request


class OutletPortal(http.Controller):

    @http.route('/outlets/public', type='http', auth='public', website=True)
    def public_outlet_list(self, **kwargs):
        outlets = request.env['hcpi.outlet.proposal'].sudo().search([('state', '=', 'active')])
        return request.render('hcpi_outlet_onboarding.public_outlet_template', {'outlets': outlets})

    @http.route('/api/v1/outlet_count', type='json', auth='user')
    def outlet_count(self):
        return request.env['hcpi.outlet.proposal'].search_count([])
```

Common reasons to add a controller:

- **Public-facing website pages** — built with the Odoo Website module.
- **Custom JSON / REST endpoints** — when XML-RPC isn't a convenient shape for an external integration.
- **Portal pages for non-internal users** — customer self-service.
- **Custom file downloads** — generated CSV/Excel with bespoke auth logic.
- **Webhooks** — external systems POSTing in to trigger something.

### Why this module doesn't have one — and what replaces it

Two reasons HCPI back-office modules almost never define their own controllers, and two replacements that cover the same ground.

**1. The web UI is generated by Odoo, not by us.** Every page you've clicked through in Parts 1–2 — proposal list, form, kanban, statusbar — is rendered by Odoo's built-in `/web` controller from the XML view definitions we wrote. We don't write our own HTTP routes for the web client; we declare what fields, views, and actions the model has, and Odoo takes care of the URLs, the rendering, the navigation. That's the whole point of being on Odoo: you get the CRUD UI for free.

**The replacement: Odoo's own `/web` controller plus the XML view system.** What you'd build with `@http.route` for a hand-rolled admin UI is replaced by `<list>`, `<form>`, `<kanban>` records plus `ir.actions.act_window` and menus. Same outcome, less code, consistent UX.

**2. The Flutter mobile app talks XML-RPC, not custom endpoints.** Enumerators in the field don't hit our URLs. The Flutter app calls Odoo over **XML-RPC** at `/xmlrpc/2/object` (and **JSON-RPC** at `/jsonrpc`). Those endpoints are part of Odoo core and they automatically expose every public method on every model. The mobile app authenticates with `/xmlrpc/2/common` (login), gets a user id, and then calls things like:

```
POST /xmlrpc/2/object
  model: 'hcpi.outlet'
  method: 'search_read'
  args: [[['active', '=', true]], ['name', 'code', 'latitude']]
```

That call hits the **exact same `search_read` you used in the Part 1 shell exercise** — Odoo's ORM is the API. No custom HTTP route on our side; Odoo's RPC layer takes care of routing, authentication, serialisation, and access-control checks (your ACL and record rules from Steps 1–3 apply to RPC calls too).

**The replacement: the built-in XML-RPC / JSON-RPC endpoint.** Any field, any model method, any computed field — the mobile app can read or call it via RPC, restricted by the same security rules that govern the web UI.

### When you *would* add a controller to HCPI

If the requirement showed up, you'd add a `controllers/` folder back. Plausible cases:

- A **public CPI dashboard** at a custom URL (`/cpi/dashboard`) that shows the latest index without logging in.
- A **webhook receiver** for external price feeds — `/hooks/prices/import`.
- A **download endpoint** for a custom binary format the QWeb PDF doesn't cover (e.g., a file for upstream statistical software).
- An **OAuth callback** for integrating a third-party identity provider.

For everything else — internal users doing work in the web UI, the mobile app talking RPC — controllers add complexity without value. That's why the folder was deleted in Step 1 and never came back.

## Exercises

1. **Restrict editing on active proposals.** Add a write-only `ir.rule` for `base.group_user` with `perm_write=True` and a domain restricting to `state == 'draft'`. Once a proposal is active, regular users can still read it but can't edit. Managers (via their own "see all" rule) keep full access.

2. **Smart button on `res.users`.** Add a `proposal_count` computed field on `res.users` (using **classical inheritance** — `_inherit = 'res.users'` with no new `_name`) and display it as a smart button on the user form via an `<xpath expr="//div[@name='button_box']" position="inside">` view inheriting `base.view_users_form`. This is the pattern country-specific modules use to extend HCPI base models — see [Country Variants](../../understanding-the-codebase/country-variants.md).

3. **Cross-record constraint.** Use `@api.constrains` on `(outlet_name, region)` to enforce "no duplicate outlet name per region" with a friendly message. Bonus: also add a `_sql_constraints` equivalent. When does each fire?

4. **Scheduled cron.** Create an `ir.cron` record that runs every Monday at 09:00 and posts a chatter message on every proposal that's been in `draft` for more than 7 days. Hint: `ir.cron` records live in data XML; their `model_id` references the model whose method runs.

5. **One controller, for real.** Add a `controllers/public_count.py` with a single `@http.route('/onboarding/active_count', type='http', auth='public')` that returns the count of active outlets as plain text. Remember to add `controllers/__init__.py` and `from . import controllers` in the module's outer `__init__.py`. Visit the URL in a browser. This is the *one* time you'll write a controller in this tutorial.

## What you learned

After Part 3 you understand the entire surface area of an HCPI module:

- **Groups, ACLs, record rules**, and how they combine.
- **HCPI's security conventions** — the `_user` / `_manager` default pair and the collector/supervisor/statistician triangle that real workflow modules use.
- **`ir.module.category`** for clean user-permissions UI.
- **`ir.rule.domain_force`** with `user.id` for "own records only" rules.
- **`@api.constrains`** vs `_sql_constraints` vs `@api.onchange` — when each fires.
- **Sequences** with `next_by_code` and a `create()` override using `@api.model_create_multi`.
- **`mail.thread`** mixin (prototype inheritance) for chatter + tracking; `mail.activity.mixin` for activities.
- **`message_post`** to write to the chatter from Python.
- **`UserError` vs `ValidationError`** and **`has_group`** for in-method permission checks.
- **What an Odoo controller is**, why most HCPI back-office modules don't need one, and the two things that replace it: Odoo's own `/web` controller (for the UI) and the built-in **XML-RPC / JSON-RPC endpoints** (for the mobile app).

## What now?

The module you built is a real module — clone it, rename it, swap the domain, and you have a working starter for almost any "an officer files, a manager approves" workflow.

From here:

- **[Module Reference](../../understanding-the-codebase/module-reference.md)** — open the real HCPI modules. You'll recognise every piece: the model files, the security files, the inheritance, the chatter, the kanbans.
- **[Country Variants](../../understanding-the-codebase/country-variants.md)** — see how Uganda, Kenya, Tanzania extend the same base modules with `_inherit` and `xpath` view inheritance.
- The remaining training topics (Coicop, Location, Outlets, Items, Data Collection, Validation, Mobile API) walk through the actual HCPI modules — everything that runs HCPI is the patterns from these three pages, applied to the price-index domain.

## Cleaning up

To remove the practice module entirely when you're done:

1. **Apps → HCPI Outlet Onboarding → Uninstall** (drops all tables and data).
2. Stop Odoo, then `rm -rf /opt/hcpi/custom/HCPI/hcpi_outlet_onboarding/`.
3. Restart Odoo.

Uninstall before deleting the folder — otherwise Odoo will warn you on every restart about the missing module that still has database records.
