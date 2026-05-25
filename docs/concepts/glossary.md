# Glossary

Quick reference for terms that appear in the codebase, the training, and the docs. Cross-cutting — covers both **CPI domain** terms and **Odoo/system** terms.

For the full explanation of how the CPI is computed, see [CPI Concepts](cpi-concepts.md).

## CPI / Statistics terms

**Basket**
The set of items a typical household consumes, used as the reference for measuring price changes. In HCPI, baskets are modelled as **consumption segments** (one per consumer group, e.g. "urban high-income", "rural"). The model is `hcpi.consumption.segment`.

**Base period (reference period)**
The period to which the current period's prices are compared. Conventionally given the value `100`. A CPI of `117.3` means current prices are 17.3% higher than the base period.

**Class** (COICOP)
The third level of the COICOP hierarchy. Sits below *Group* and above *Sub-class*. Model: `hcpi.class`. Example: "01.1.1 — Bread and cereals."

**COICOP**
Classification of Individual Consumption by Purpose. The international standard for categorizing household consumption, published by the UN Statistics Division. HCPI uses all six levels: Division → Group → Class → Sub-class → Micro-class → Elementary aggregate. Owned by the `hcpi_coicop` module.

**Composite weight**
The weight assigned to a level of the COICOP hierarchy, representing that category's share of household spending. Each higher level's composite weight is the sum of its children's. Field name: `composite_weight` on every COICOP model.

**Consumption segment**
HCPI's term for a basket. A grouping of similar consumers used to compute basket-specific CPIs. Model: `hcpi.consumption.segment`.

**CPI (Consumer Price Index)**
A single number measuring the cost of a typical household's consumption against a base period. The headline output of the HCPI system.

**Division** (COICOP)
The top level of the COICOP hierarchy. There are 12-14 of them in standard COICOP, like "01 — Food and non-alcoholic beverages". Model: `hcpi.division`.

**Domestic index**
An index restricted to domestic (non-imported) items or markets. Model: `hcpi.domestic.index`.

**EA**
See *Elementary aggregate*.

**Elementary aggregate (EA)**
The deepest (sixth) level of the COICOP hierarchy. Items within one EA are treated as substitutes for each other — averaged using a simple geometric mean, with no further weighting below the EA level. The base for item creation. Model: `hcpi.elementary.aggregate`.

**Enumerator**
The person who visits outlets to record prices. Called `data_collector` in the codebase (`data_collector_id` on `hcpi.data.collection`).

**Geometric mean**
The Nth root of the product of N values. Used below the EA level to average price relatives without weighting. Robust to outliers compared to arithmetic mean.

**Group** (COICOP)
The second level of the COICOP hierarchy. Sits below *Division* and above *Class*. Model: `hcpi.group`. Example: "01.1 — Food."

**Index**
A value computed from price relatives at some level of aggregation (per basket, per COICOP level, per month). In HCPI, every model with `.index` in its name is a stored computed index. The headline CPI is `hcpi.national.index`.

**Inlier**
A price observation **not** flagged as an outlier by the Tukey algorithm — a "normal" price. Field: `is_inlier` (computed) on `hcpi.data.collection.line`.

**IQR (Interquartile range)**
The difference between the third quartile (Q3) and the first quartile (Q1) of a dataset. Used by the Tukey algorithm: any observation outside `[Q1 − k·IQR, Q3 + k·IQR]` is an outlier.

**Item**
A specific good or service whose price is collected (e.g. "white rice, 1 kg, retail"). Belongs to one elementary aggregate. Model: `hcpi.item`.

**Laspeyres index**
A CPI formula that uses base-period quantities as weights. The most common CPI methodology globally. HCPI's index computation follows this approach (weighted by base-period basket composition).

**Long-term price relative**
The ratio of the current price to the base-period price. The unit of CPI computation. Stored on `hcpi.outlet.item.observation`.

**Median**
The middle value when a dataset is sorted. Robust to outliers. Used by the Tukey algorithm.

**Micro-class** (COICOP)
The fifth level of the COICOP hierarchy. Sits below *Sub-class* and above *Elementary aggregate*. Model: `hcpi.micro.class`.

**National index**
The headline CPI value — the index computed nationally across all baskets and all COICOP levels. Model: `hcpi.national.index`. Unique per `(coicop, month)`.

**Observation**
A single recorded price — what an enumerator types into the questionnaire. "On date D at outlet O, item I cost X". Model: `hcpi.outlet.item.observation`. Many millions accumulate over the system's lifetime.

**Outlet**
A physical or logical point where prices are collected — a shop, a market stall, a supermarket. Model: `hcpi.outlet`.

**Outlet item**
A junction record linking one outlet to one item it sells, with the item's `base_price` at that outlet and the time series of observations. Model: `hcpi.outlet.item`.

**Outlet type**
A categorization of outlets — supermarket, retail, wholesale, market stall. Model: `hcpi.outlet.type`.

**Outlier**
A price observation flagged as anomalous by the Tukey algorithm. Field: `is_outlier` (computed) on `hcpi.data.collection.line`. Reviewed by supervisors during the validation state of the workflow.

**Price relative**
A ratio: current price ÷ reference price. CPIs are built by aggregating price relatives, not raw prices. HCPI stores both *long-term* (vs. base period) and *short-term* (vs. previous month) relatives.

**Questionnaire**
The collection of price observations for one outlet visit. Model: `hcpi.data.collection`. Has a state machine: draft → survey → standardization → validation → done.

**Short-term price relative**
The ratio of the current price to the previous month's price. Used for month-over-month change detection and outlier flagging. Stored on `hcpi.outlet.item.observation`.

**Standardization**
The step in the workflow where observed quantities (e.g. "350 g sachet") are normalized to the item's standard unit (e.g. "per kg"). A state in the `hcpi.data.collection` state machine.

**Sub-class** (COICOP)
The fourth level of the COICOP hierarchy. Sits below *Class* and above *Micro-class*. Model: `hcpi.sub.class`.

**Supervisor**
The person who reviews and validates questionnaires after the enumerator collects them. Field: `data_supervisor_id` on `hcpi.data.collection`. Belongs to the `group_data_collection_supervisor` access group.

**Tukey algorithm**
A robust outlier-detection method using the median and interquartile range. Flags as outliers any observations outside `[Q1 − k·IQR, Q3 + k·IQR]` where `k` is typically 1.5 or 3. HCPI's primary outlier filter.

**UoM (Unit of measure) / UoO (Unit of observation)**
The units in which prices are recorded — kg, litre, piece, packet. Model: `hcpi.uoo` (Unit of Observation).

**Weights** (CPI)
Numbers attached to COICOP categories representing their share of household spending. Derived from expenditure surveys. Without correct weights, the CPI doesn't reflect real consumer experience.

**Zero-price observation**
An observation with a recorded price of zero. Usually means the item was out of stock, not actually free. HCPI's `hcpi_computation` mixin checks that there are at least 6 safe months and no more than 5 problematic items before allowing index computation.

---

## Odoo / system terms

**Action** (Odoo)
A record describing what to do when a menu or button is clicked. Most common type: `ir.actions.act_window` (open a list/form for a model). See [Odoo Basics](../understanding-the-codebase/odoo-basics.md#actions-what-happens-when-you-click).

**Addons path**
The `addons_path` setting in `hcpi.conf` — a comma-separated list of folders where Odoo looks for modules. Each folder is expected to *contain* module folders.

**Admin password (master password)**
The `admin_passwd` value in `hcpi.conf`. Used for database-level operations (create, drop, backup) through Odoo's web DB manager. **Not** the same as a user login password. See [User Administration](../training/day1/user-administration.md).

**Developer mode**
Odoo UI mode that exposes technical info (model names, view IDs) for diagnostic and developer use. Activate via Settings → "Activate the developer mode" or append `?debug=1` to a URL.

**Domain**
Odoo's filter expression syntax — e.g. `[('active', '=', True), ('district_id.name', '=', 'Kampala')]`. Translates to SQL WHERE clauses via the ORM.

**Field** (Odoo)
A column on a model — i.e. a Python attribute on the model class that becomes a DB column. Many field types: `Char`, `Integer`, `Many2one`, `One2many`, `Many2many`, etc.

**Form view**
A view that shows one record laid out for editing. XML tag: `<form>`.

**Group** (Odoo security)
A permission group. Users belong to groups; groups have CRUD permissions on models (via `ir.model.access.csv`) and can be restricted from menus. Examples: `coicop_user`, `coicop_manager`, `group_data_collection_supervisor`.

**ir.actions.act_window**
The Odoo model storing "open a list/form view" actions. Stored in DB; populated from XML records on module install/upgrade.

**ir.cron**
Odoo's scheduled-job table. Used by the secretariat hub to schedule data syncs from country instances.

**ir.model.access.csv**
A CSV file in each module's `security/` folder defining CRUD permissions (`perm_read`, `perm_write`, `perm_create`, `perm_unlink`) for each model × group combination.

**ir.rule**
The Odoo model storing record-level access rules. A rule is a domain that automatically filters queries against a model for a given group. Used for "users can only see records in their region" patterns.

**ir.ui.view**
The Odoo table storing all views. Populated from XML records on module install/upgrade.

**List view (tree view)**
A view showing many records as a table. XML tag: `<list>` (modern) or `<tree>` (older — same thing).

**`_inherit`**
A Python attribute on an Odoo model that says "extend this existing model" rather than create a new one. Used heavily by HCPI's country overlays (e.g. `ug_outlet` extends `hcpi.outlet`).

**Manifest** (`__manifest__.py`)
The metadata file at the root of every Odoo module. Declares the module's name, dependencies, version, data files, and assets.

**Menu**
A clickable label in the navigation. Model: `ir.ui.menu`. Populated from `<menuitem>` records in XML.

**Model** (Odoo)
A Python class declaring an `_name` attribute. Becomes a database table; each instance is a record.

**Module** (Odoo)
A folder under `addons_path` containing `__manifest__.py`, Python models, XML views, security, etc. The smallest unit of Odoo deliverable functionality.

**ORM**
Odoo's Object-Relational Mapper. The Python interface for reading and writing data without raw SQL. Core methods: `search`, `browse`, `read`, `create`, `write`, `unlink`.

**OWL**
Odoo's frontend JavaScript framework, similar in spirit to React. Used by `hcpi_dashboard` and the secretariat hub for interactive UI components.

**queue_job**
Third-party module (OCA) that runs Python jobs asynchronously in background workers. HCPI uses it for long-running index computations. Configured in the `[queue_job]` section of `hcpi.conf`.

**Recordset**
A list-like Odoo object holding zero or more records of the same model. The return type of `search()` and `browse()`.

**Search view**
A view defining the search bar's filters and group-by options. XML tag: `<search>`.

**`server_wide_modules`**
A `hcpi.conf` setting listing modules that must be loaded into the Odoo process at startup, before any database is opened. For HCPI: `base,web,queue_job`.

**`-u <module>`**
Command-line flag for `odoo-bin`. Tells Odoo to *upgrade* a module — re-read its XML, run any migrations, refresh views. Used after editing source files.

**`--dev=xml`** (or `--dev=all`)
Command-line flag that makes Odoo re-read XML view files on every request. Lets you iterate on view changes without restarting. `--dev=all` adds Python module reloading and other dev affordances.

**Wizard** (Odoo)
A transient model (`models.TransientModel`) used to drive multi-step user interactions — typically a form that collects inputs, runs an action, then disappears. Heavily used in `hcpi_index` for triggering computations.

**`web_progress`**
Third-party module providing UI for long-running operations — a modal with progress percentage and a "run in background" button. Wired into the index-computation wizards.

**XML-RPC**
The protocol the EAC Secretariat hub uses to pull CPI snapshots from country instances. Odoo exposes every model method over XML-RPC by default.

---

## HCPI-specific terms

**EA-level computation**
Computation of indices at the elementary-aggregate level — the bottommost layer above which weighting kicks in. Model: `hcpi.basket.elementary.aggregate.index`. Triggered by the `ea.index.update` wizard.

**`hcpi-files.zip`**
The archive produced by the [extraction guide](../extraction/linux-export.md) containing a country's `conf/` and `custom/` folders. The deployable artifact for installing HCPI on a new machine.

**`hcpi.dump`**
A PostgreSQL custom-format database dump produced by `pg_dump -Fc`. Restored with `pg_restore`. Part of the extraction artifact set.

**HCPI**
Harmonized Consumer Price Index. The product. Also the name of the Odoo module set that implements it.

**Master password**
See *Admin password*.

**Mtnce branch**
Maintenance branch — e.g. `ke_18_mtnce`, `tz_18_mtnce`. Holds the released/maintained variant of a country branch.

**Outlet code**
The string identifier on `hcpi.outlet`, validated against a strict format like `255.53.21.03.003`.

**Overlay module**
A country-specific module (`ug_*`, `ke_*`, `tz_*`, `zar_*`) that extends the shared `hcpi_*` core. See [Country Variants](../understanding-the-codebase/country-variants.md).

**ZAR** (in this codebase)
Zanzibar, **not** South Africa or the South African Rand. The `zar_*` modules customize HCPI for Zanzibar. Easy to confuse — be careful.
