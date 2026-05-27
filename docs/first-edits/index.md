# Making Your First Edits

This page walks you through the smallest possible change to HCPI: renaming the **Outlets** menu in the app launcher. It takes about five minutes, touches one line of XML, and the result is immediately visible in your browser.

The point isn't the change itself — it's the **edit → upgrade → refresh** loop. Get this loop comfortable and every more interesting change becomes easy.

!!! info "Before you start"
    - HCPI is installed and you can reach `http://localhost:9201` in a browser. If not, finish the [installation guide](../installation/windows-wsl.md) first.
    - You've read [Odoo Basics](../understanding-the-codebase/odoo-basics.md) — you know what a menu, action, and view are.
    - VS Code (or PyCharm) is open on `/opt/hcpi/` with the venv selected.

## The change we're making

Right now your app launcher has an **Outlets** tile (visible on the home screen and in the sidebar). We're going to rename it to **Outlets & Markets**.

That's it. One word changed. Then we'll iterate — change the empty-state message, reorder a column — to build the muscle memory for bigger changes.

## Step 1: Find the file

You already know from [Odoo Basics](../understanding-the-codebase/odoo-basics.md) that menus live in XML files. From the [Module Reference](../understanding-the-codebase/module-reference.md) you know that outlets are owned by `hcpi_outlet`. So the file you want is somewhere under `custom/HCPI/hcpi_outlet/views/`.

In VS Code, open `custom/HCPI/hcpi_outlet/views/hcpi_outlet_menus.xml`. The top of the file looks like this:

```xml
<menuitem id="menu_hcpi_outlet_root"
          name="Outlets"
          web_icon="hcpi_outlet,static/description/icon.png"
          sequence="25"
          groups="hcpi_outlet.group_outlet_manager" />
```

That `name="Outlets"` is what shows up in the app launcher.

!!! tip "How would you have found this without being told?"
    Two ways:

    1. **In the UI** — turn on Developer Mode (top-right user menu → Settings → Activate the developer mode). Then hover the **Outlets** menu in the sidebar: a tooltip shows the menu's XML ID, `hcpi_outlet.menu_hcpi_outlet_root`. Grep your project for `menu_hcpi_outlet_root` and you'll land on this file.
    2. **In the IDE** — open project-wide search (Ctrl+Shift+F in VS Code), search for `name="Outlets"` inside `custom/HCPI/`. Three hits — the root menu, a sub-menu, and an action's name.

## Step 2: Make the change

Change line 5 from:

```xml
name="Outlets"
```

to:

```xml
name="Outlets &amp; Markets"
```

(`&amp;` is XML-speak for `&` — XML treats raw `&` as a parse error, which is one of the first papercuts you hit when working with view files.)

Save the file.

## Step 3: Apply the change

Editing the file does *nothing* on its own. Odoo reads menus out of the database — specifically out of `ir.ui.menu`, as you saw in [Odoo Basics](../understanding-the-codebase/odoo-basics.md). You need to tell Odoo to re-read the XML and overwrite the existing records.

Two ways. Pick one:

### Option A: Restart with `-u hcpi_outlet` (works always)

Stop Odoo (`Ctrl+C` in the terminal where it's running), then start it again with the upgrade flag:

```bash
cd /opt/hcpi
source venv/bin/activate
python odoo/odoo-bin -c conf/hcpi.conf -u hcpi_outlet
```

`-u hcpi_outlet` tells Odoo: "before serving requests, re-install the `hcpi_outlet` module — re-load its XML, re-create any tables that have changed, etc." It takes a few seconds.

Wait for the line `HTTP service (werkzeug) running on ... port 9201`. That's your signal.

### Option B: Run with `--dev=xml` (faster for iteration)

If you start Odoo with `--dev=xml`, it re-reads XML view files on every request without needing a restart:

```bash
python odoo/odoo-bin -c conf/hcpi.conf --dev=xml
```

Now you can edit any XML view and just refresh the browser — no restart needed. **Caveat:** this only covers XML view re-reading. If you change Python code, you still need to restart. Stick with `--dev=xml` for the iterations below.

!!! warning "`--dev=xml` doesn't reload menus reliably"
    Menus are mostly cached in the user's session. If your menu rename doesn't appear with `--dev=xml`, hard-refresh (`Ctrl+Shift+R`) and log out/in. If it still doesn't appear, fall back to Option A — `-u hcpi_outlet` will definitely apply it.

## Step 4: See the change

Open `http://localhost:9201` in your browser. Hard-refresh (`Ctrl+Shift+R`).

You should see **Outlets & Markets** wherever you saw **Outlets** before — in the app launcher tile and in the sidebar.

If you don't:

- **Didn't refresh hard enough.** `Ctrl+Shift+R` (or open DevTools and disable cache).
- **Used `--dev=xml` and it didn't pick up.** Stop Odoo and restart with `-u hcpi_outlet`.
- **Wrong file.** You changed `menu_hcpi_outlet`, not `menu_hcpi_outlet_root`. There are two menus named "Outlets" in this file — the root (in the app launcher) and a sub-menu (inside the root). Rename the *root* one to see the change in the app launcher.
- **XML parse error.** Did you escape the `&` as `&amp;`? Check the Odoo terminal output for a parse error.

That's the full loop. Edit → upgrade → refresh → verify.

---

## Iteration 2: Change the empty-state message

Now you've done it once, here's a second change in the same module. This time we'll touch an **action** instead of a menu.

Open `custom/HCPI/hcpi_outlet/views/hcpi_outlet_views.xml`, scroll to the bottom (around line 216), and find:

```xml
<record id="hcpi_outlet_action" model="ir.actions.act_window">
    <field name="name">Outlets</field>
    <field name="res_model">hcpi.outlet</field>
    <field name="view_mode">list,form</field>
    <field name="domain">[]</field>
    <field name="help" type="html">
        <p class="o_view_nocontent_smiling_face">Create an Outlet!</p>
    </field>
</record>
```

Change the help text:

```xml
<p class="o_view_nocontent_smiling_face">No outlets yet. Click "New" to add the first one.</p>
```

Save. If you're running `--dev=xml`, just refresh the browser. Otherwise, restart with `-u hcpi_outlet`.

To see the result, you need the outlet list to be **empty** — Odoo shows this help text only when there are no records to display. If you have outlets, you can either temporarily archive them all, or apply a filter that excludes everything (search "Without Items" or type a nonsense name into the search box).

## Iteration 3: Reorder columns in the list view

Open the same file (`hcpi_outlet_views.xml`) and find the list view (around line 199):

```xml
<list string="">
    <field name="name" optional="show" />
    <field name="outlet_code" optional="show" />
    <field name="address" optional="hide" />
    <field name="consumption_segment_id" optional="show" />
    <field name="outlet_type_id" optional="show" />
    <field name="latitude" optional="hide" />
    <field name="longitude" optional="hide" />
    <field name="description" optional="hide" />
    <field name="outlet_item_count" optional="show" />
</list>
```

This is the list view definition. Each `<field>` is a column.

- `optional="show"` — visible by default, but the user can hide it via the column toggle.
- `optional="hide"` — hidden by default, but the user can turn it on.
- No `optional=` — always visible, can't be hidden.

Try this: put `outlet_code` first (before `name`).

```xml
<list string="">
    <field name="outlet_code" optional="show" />
    <field name="name" optional="show" />
    <field name="address" optional="hide" />
    <field name="consumption_segment_id" optional="show" />
    <field name="outlet_type_id" optional="show" />
    <field name="latitude" optional="hide" />
    <field name="longitude" optional="hide" />
    <field name="description" optional="hide" />
    <field name="outlet_item_count" optional="show" />
</list>
```

Save → refresh (or `-u hcpi_outlet`) → look at the outlet list. The code column is now leftmost.

!!! tip "Try `optional="hide"` on `outlet_item_count`"
    Set `optional="hide"` on `outlet_item_count` and refresh. The column disappears from the default view. Click the column toggle (top-right of the list) and you can bring it back.

## Iteration 4: Add a constant field to the form

This one's slightly bigger — you'll change the form view, not the list. Open `hcpi_outlet_views.xml`, find the form view (around line 35), and look at the main group:

```xml
<group>
    <group>
        <field name="outlet_code" />
        <field name="outlet_type_id" />
        <field name="consumption_segment_id" />
        <field name="address" />
    </group>
    <group>
        <field name="description" />
    </group>
</group>
```

The outer `<group>` is a two-column layout. Each inner `<group>` is one column. Add a field that already exists on the model — say `latitude` — to the right column:

```xml
<group>
    <group>
        <field name="outlet_code" />
        <field name="outlet_type_id" />
        <field name="consumption_segment_id" />
        <field name="address" />
    </group>
    <group>
        <field name="description" />
        <field name="latitude" />
    </group>
</group>
```

Save → refresh → open any outlet. `Latitude` now appears under `Description` in the right column of the form.

(Notice we didn't have to touch the model — `latitude` is already defined in `hcpi_outlet.py`. Adding it to a view just makes it visible. Adding a *new* field would require both a model edit and a view edit — that's the next step beyond this page.)

## What you learned

In about half an hour:

- The location of menus, actions, and views inside an HCPI module
- The edit → upgrade → refresh loop (and the `--dev=xml` shortcut for view changes)
- How to map a UI element to its XML definition via Developer Mode and project-wide search
- The difference between modifying a **menu** label (`<menuitem name="..."/>`), an **action** field (`<field name="name">...</field>`), and a **view** structure (`<field name="latitude"/>` inside a `<group>`)

## What's next

The exercises above were view-only — XML on disk, no Python. The next levels involve editing the model:

1. **Add a new field** — give `hcpi.outlet` a `notes` field of type `Text`, then show it on the form. This teaches model + view changes together, including the database migration that runs on `-u hcpi_outlet`.
2. **Add a computed field** — a field whose value comes from a Python method (e.g., `is_remote` based on lat/long). Teaches `@api.depends`.
3. **Add a button with an action** — put a button on the form that, when clicked, calls a Python method. Teaches how UI events reach Python.
4. **Create a small report** — a one-page PDF showing an outlet's details. Teaches QWeb templating.

These will be written up in upcoming sections.

For now, the natural next step is **[Your First Module](your-first-module.md)** — build a complete, minimal Odoo module from scratch (model, views, action, menu, security) and uninstall it cleanly when you're done.

## Reverting your changes

When you've finished playing, revert if you don't want to keep the edits:

```bash
cd /opt/hcpi/custom/HCPI/hcpi_outlet
git status                          # see what you changed
git diff                            # see the actual diff
git checkout -- views/              # discard view changes
```

(If `custom/HCPI/` isn't a git repo on your machine, no harm done — your changes just stay in place until you change them back manually.)

Then restart Odoo once more with `-u hcpi_outlet` to push the original XML back into the database.
