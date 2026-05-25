# Development Environment

You can edit HCPI with `nano` if you want, but you won't enjoy it. This page sets up a proper IDE so the rest of the week is comfortable.

## What we're aiming for

By the end of this step:

- A code editor open on `/opt/hcpi/` with project-wide search, file navigation, and Python autocomplete
- The editor pointing at the project venv (so imports resolve and type hints work)
- The official **Odoo extension** installed for navigation between models, fields, and views

## Why an IDE matters for HCPI

Odoo's pattern is to spread one logical feature across several files. An outlet, for example, is:

- A model in `hcpi_outlet/models/outlet.py`
- A form view in `hcpi_outlet/views/outlet_views.xml`
- Access rules in `hcpi_outlet/security/ir.model.access.csv`
- Possibly an icon in `hcpi_outlet/static/description/icon.png`

When you want to understand what happens on a click, you'll jump between these files dozens of times. A good IDE makes that:

- **Go-to-definition** — click on `self.env['hcpi.outlet']` and jump to the model
- **Find references** — see everywhere a field is read or written
- **Project-wide search** — grep across all modules in milliseconds
- **Autocomplete** — Odoo's ORM methods (`search`, `read`, `create`, `write`) are massive and well-typed

## VS Code (recommended for WSL)

VS Code is free, has first-class WSL integration, and has the most-maintained Odoo extension. It's the recommended option for training.

What you'll install:

1. **VS Code** on Windows ([code.visualstudio.com](https://code.visualstudio.com/)) — with "Add to PATH" ticked during installation
2. **WSL extension** (Microsoft) — lets VS Code connect into your WSL filesystem
3. **Python extension** (Microsoft) — autocomplete, debugger, interpreter selection
4. **Odoo extension** (Odoo S.A.) — navigation for models, fields, XML views

The Odoo extension is what makes this setup specifically good for HCPI work — it knows what `@api.depends` means, can follow `_inherit`, jumps from a view's XML to the model it's bound to.

### How the WSL integration works

VS Code itself runs on Windows. When you "Open Folder" into a WSL path, VS Code spawns a small server *inside* WSL that does the file reading, language analysis, and terminal — the Windows-side VS Code is just the UI. This means:

- File watching is fast (Linux filesystem, not the slow `/mnt/c` bridge)
- The integrated terminal opens inside WSL by default — no manual `wsl` command
- The Python interpreter you pick is the WSL one (`/opt/hcpi/venv/bin/python3`)

You launch it from inside WSL:

```bash
cd /opt/hcpi
code .
```

If you see "command not found", restart your WSL terminal — VS Code's `code` shim only appears in shells started after the WSL extension installs.

### Selecting the interpreter

Once VS Code is open on the project, the next critical step is telling it about the venv. Without this, you'll see red squiggles everywhere because VS Code is checking against the system Python, which doesn't have `werkzeug` or `odoo`.

`Ctrl+Shift+P` → `Python: Select Interpreter` → pick `./venv/bin/python3` (or browse to `/opt/hcpi/venv/bin/python3`).

The status bar at the bottom-left should now show the venv path. Imports will resolve.

## PyCharm (alternative)

PyCharm is a fine choice if you already know it. Caveats specifically for WSL development:

- **WSL remote support is Professional-only** (~$99/year). The free Community edition has to access WSL files via the `\\wsl$\Ubuntu\opt\hcpi` network path, which is slower than VS Code's approach.
- **Database tools** (often handy for HCPI work — running ad-hoc queries against PostgreSQL) are also Professional-only.

If you have a Professional licence, it's a great experience. Otherwise, VS Code is the better fit for WSL.

Configuration is conceptually the same: open the project, set the interpreter to the venv's Python.

## "I'm working on a real Linux server, not WSL"

Same advice but simpler. Options:

- **VS Code Remote-SSH** — VS Code on your laptop, files on the server. Same architecture as WSL Remote.
- **VS Code on the server, in a browser** — install `code-server` if you want a browser-based editor on the server itself.
- **Vim/Emacs** — if you already live there, you already know what you're doing.

The point either way is the same: editor with project-wide search, Python autocomplete, and (ideally) Odoo-aware navigation.

## A short list of habits that help

Once your IDE is set up:

- **Use project search aggressively.** When you see `hcpi.outlet` or `outlet_form_view`, grep for it across `custom/HCPI/`. This is how you learn the codebase.
- **Bookmark `models/` and `views/` folders.** You'll be in them constantly.
- **Use Developer Mode in the browser** alongside the IDE. With developer mode on, hovering a field shows its technical name; you grep that name in the IDE to find the file. (Covered on [User Administration](user-administration.md) and [Understanding the Codebase](../../understanding-the-codebase/index.md).)
- **Keep a second terminal open** running `tail -f /opt/hcpi/log/hcpi.log` — Odoo's log is where errors show up first.

## Practical steps

The WSL guide has the most detailed walkthrough including screenshots:

- **[Windows WSL](../../installation/windows-wsl.md)** — Step 9 (VS Code + WSL + Python + Odoo extension, with the PyCharm alternative)

➡️ Next: [Configuration File](configuration.md)
