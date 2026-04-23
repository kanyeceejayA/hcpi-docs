# Making Your First Edits

This section is a runway of small, concrete changes. Each one teaches you one idea and has an obvious "did it work?" check in the browser.

!!! info "Before you start"
    Complete [Understanding the Codebase](../understanding-the-codebase/index.md) first — you'll need to know how to map what you see in the browser to the file that produces it.

## The loop you'll use for every change

Every code change in Odoo follows roughly the same cycle:

1. **Edit** the file in your IDE.
2. **Restart Odoo with `-u <module>`** to upgrade that one module. Example: `python odoo/odoo-bin -c conf/hcpi.conf -u hcpi_outlet --stop-after-init`. This reloads the module's Python, views, and data into the running database.
3. **Refresh your browser.** Hard-refresh (`Ctrl+Shift+R`) to bypass any cached assets.
4. **Check the change is visible.** If it's not, read the Odoo terminal output for errors.

Most changes need step 2. Pure XML view changes sometimes pick up without restart if you have `--dev=xml` running; Python changes always need a restart.

## Planned exercises

The following exercises will be fleshed out in upcoming sections. For now, here's the shape of the series so you know what's coming:

1. **Rename a visible label** — change the text "Outlet" to "Shop" (or similar) in one view. Smallest possible change; teaches the edit → upgrade → refresh loop.
2. **Add a simple field** — add a "Notes" text field to the outlet model and show it on the form. Teaches how models and views interact.
3. **Add a computed field** — add a field that displays something calculated from other fields. Teaches Python-side logic.
4. **Add a button with an action** — put a button on the form that, when clicked, does something (like toggling a status). Teaches how UI events reach Python code.
5. **Create a small report** — a short custom report (PDF or screen). Teaches how reports fit into Odoo.

Each exercise will link back to the relevant concepts from [Understanding the Codebase](../understanding-the-codebase/index.md).
