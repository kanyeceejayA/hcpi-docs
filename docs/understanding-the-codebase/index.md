# Understanding the Codebase

Now that HCPI is running on your machine, this section will help you find your way around the code. Our goal here isn't to make you an Odoo expert in one sitting — it's to give you a map so that when you look at a folder or file, you know roughly what it's for and where to change things.

We'll move from big picture down to detail:

1. **The folders on your disk** — what each top-level folder is for
2. **Anatomy of an HCPI module** — what's inside `custom/HCPI/*` and why
3. **Matching the UI to the code** — when you see something in the browser, how do you find the file that produces it?

## Your install layout

When you finished the installation guide, you ended up with something like this (paths are Linux/WSL — Windows has the same shape with backslashes):

```
/opt/hcpi/
├── conf/          ← HCPI-specific configuration
├── custom/        ← HCPI's own code (the Uganda-specific modules)
│   └── HCPI/
├── log/           ← Runtime log files
├── odoo/          ← Odoo 18 framework (cloned from GitHub)
│   ├── addons/    ← Odoo's built-in modules (mail, web, etc.)
│   ├── odoo/      ← The Odoo core (not usually touched)
│   └── odoo-bin   ← The entry-point script you run
└── venv/          ← Python virtual environment (dependencies)
```

Three of these folders matter day-to-day:

| Folder | What you'll do here |
|---|---|
| `conf/` | Change how HCPI runs (ports, DB connection, addon paths) |
| `custom/HCPI/` | **This is where HCPI's logic lives** — the folder you'll edit most |
| `odoo/` | Look up how built-in Odoo features work (read-only in practice) |

The `odoo/` folder is big and slightly intimidating, but you'll mostly just *read* from it — to understand how a built-in field or method works so you can use or extend it. You won't modify it; changes there get overwritten the next time you update Odoo.

!!! info "Placeholder: folder tree screenshot"
    *(A screenshot of the folder tree as it appears in VS Code's file explorer, with `custom/HCPI/` highlighted, will go here.)*

## Anatomy of an HCPI module

Open `custom/HCPI/` in your IDE. You'll see a list of folders like:

```
custom/HCPI/
├── hcpi_brand/
├── hcpi_coicop/
├── hcpi_computation/
├── hcpi_dashboard/
├── hcpi_data_collection/
├── hcpi_item/
├── hcpi_outlet/
├── queue_job/
├── ug_base_import/
├── ug_location/
├── ug_outlet/
└── ... (more)
```

**Each folder is its own "module".** A module is the smallest unit of functionality in Odoo — it bundles together database models, user interface, access rules, and data for one coherent feature. For example, `hcpi_outlet/` is everything about outlets (shops where prices are collected); `hcpi_dashboard/` is everything about the dashboard.

Modules with the `ug_` prefix are Uganda-specific; others are generic HCPI. Countries adopting HCPI may replace or add to the `ug_*` modules with their own.

### Inside one module

Open any module folder, e.g. `hcpi_outlet/`. A typical module has this layout:

```
hcpi_outlet/
├── __init__.py            ← Makes Python treat this as a package
├── __manifest__.py        ← The module's "identity card" (metadata)
├── models/                ← Python files that define database tables
│   ├── __init__.py
│   └── outlet.py
├── views/                 ← XML files that define the UI
│   └── outlet_views.xml
├── security/              ← Who can see/edit what
│   ├── ir.model.access.csv
│   └── outlet_security.xml
├── data/                  ← Seed data loaded at install time
│   └── outlet_data.xml
├── static/                ← JavaScript, CSS, images
│   └── description/
│       └── icon.png
└── i18n/                  ← Translations
    └── fr.po
```

Here's what you'd change in each:

| Folder/file | When you change it |
|---|---|
| `__manifest__.py` | Changing version, dependencies, author, or what other files to load |
| `models/` | Adding a new database field, writing a calculation, validating input |
| `views/` | Changing what a form/list looks like, adding buttons, reorganizing fields |
| `security/` | Granting or restricting access to a model for a user group |
| `data/` | Pre-filling a table with default records at install time |
| `static/` | Adding an icon, a stylesheet, a piece of JavaScript |
| `i18n/` | Adding or updating a translation |

Most day-to-day work happens in `models/` and `views/`. Security is critical but rarely changes once set up.

!!! info "Placeholder: module structure screenshot"
    *(A screenshot of `hcpi_outlet/` expanded in VS Code's file explorer, with each subfolder labeled with a short description, will go here.)*

## Matching the UI to the code

When you see a feature in the browser and want to know "where is this defined?", here's the general flow:

1. **Turn on Developer Mode.** Top-right user menu → Settings → in the Settings app → click "Activate the developer mode". A small bug icon appears in the top bar.
2. **Hover over any field.** With developer mode on, hover a field label in a form view; a tooltip shows you the **technical name** of the model (e.g. `hcpi.outlet`) and field (e.g. `name`).
3. **Use that technical name to find the file.** If the model is `hcpi.outlet`, it's defined in a Python file somewhere under `custom/HCPI/`. Grep for it: `grep -r "_name = 'hcpi.outlet'" custom/HCPI/`.
4. **Views work the same way.** With developer mode, you can open the "Edit Form View" dialog (Developer menu → Edit View → Form) which shows you the view's XML ID. Grep for that ID in the `views/` folders.

!!! info "Placeholder: developer mode screenshots"
    *(Two screenshots go here: (1) the Settings page showing the "Activate the developer mode" link; (2) a form view with developer mode on, hovering a field to show the tooltip with the technical name.)*

### A worked example

Let's say you want to find where "Outlet Name" is displayed on the outlet form.

1. Go to Field Operations → Outlets → pick any outlet to open its form.
2. With developer mode on, hover the "Name" label — the tooltip shows model `hcpi.outlet`, field `name`.
3. In your IDE, grep for `_name = 'hcpi.outlet'`. You'll find it in `custom/HCPI/hcpi_outlet/models/outlet.py`.
4. For the view, Developer menu → Edit View → Form shows you the view's XML ID, e.g. `hcpi_outlet.outlet_form_view`. That's defined in `custom/HCPI/hcpi_outlet/views/outlet_views.xml`.

Now you know exactly which two files to edit if you want to change how "Outlet Name" looks or behaves.

## What's next?

Ready to make your first change? Continue to **[Making Your First Edits](../first-edits/index.md)** — a short series of small, safe changes that build your confidence step by step.
