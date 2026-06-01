# Infrastructure & Third-party

A handful of modules HCPI **vendors** rather than writes — pulled in to keep installs reproducible and not depend on hunting for the right OCA branch. Treat them as **read-only** unless you have a specific reason to patch.

Even read-only, you'll occasionally need to know which one is responsible for what — the page below maps each module to "what it does" and "where to look if it's involved in a problem."

## `queue_job` — OCA's job queue framework

**What it does:** Runs long-running operations asynchronously in background worker processes. Index computation, bulk import, and anything that takes more than a few seconds is dispatched through `queue_job` instead of blocking the HTTP request.

**Why HCPI uses it:** computing an EA-level index over a year of data can take minutes. Doing that in a web request would time out the browser and tie up the worker. `queue_job` puts it in a queue, a separate worker picks it up, and the user gets a progress bar (via `web_progress`).

**Where it shows up in HCPI:**

- Every index wizard in [`hcpi_index`](indices-dashboard.md) dispatches its computation through `queue_job`.
- The `[queue_job]` section of `hcpi.conf` configures channels (how many parallel jobs of each type can run).

**Critical config in `hcpi.conf`:**

```ini
server_wide_modules = base,web,queue_job

[queue_job]
channels = root:3,root.hcpi_ea:1
scheme = http
host = localhost
port = 9201
```

`server_wide_modules` is what makes `queue_job` load **before any database is opened** — it needs to start workers before requests can be served. If you forget this, jobs hang at "enqueued" and never run.

**Where to look to change something:**

| You want to… | Open |
|---|---|
| Run more parallel index jobs | `hcpi.conf` `[queue_job]` `channels` — increase the number after `:` |
| Throttle a specific job type | Add a sub-channel like `root.hcpi_ea:1` and reduce it to `1` |
| Inspect failed jobs | UI → Settings → Technical → Job Queue. Failed jobs show their traceback. |
| Re-run a failed job | Same UI — open the job, click **Requeue**. |
| Debug "job never ran" | Confirm `queue_job` is in `server_wide_modules`. Check the worker process is running (`ps aux | grep queue_job`). |

**Reference:** <https://github.com/OCA/queue>.

## `web_progress` — progress bars for long jobs

**What it does:** A UI overlay for long-running operations — a modal with a progress percentage, an ETA, and a "run in background" button that lets the user keep using the UI while the job continues.

**Why HCPI uses it:** the index wizards take time. Without `web_progress` the user stares at a spinner with no idea whether progress is happening.

**Where it shows up in HCPI:**

- The index-computation wizards in [`hcpi_index`](indices-dashboard.md) all integrate with `web_progress`.
- Some bulk-update operations on `hcpi.data.collection`.

**Where to look to change something:**

| You want to… | Open |
|---|---|
| Add progress to a new long operation | Wrap the loop with `with progress(...)` calls — see existing usage in `hcpi_index/wizards/` |
| Change the progress-bar colour | See `progress_bar_customization` below |
| Hide the "run in background" button | Override the JS template in your own module's `assets` |

**Reference:** <https://github.com/OCA/web>.

## `kola_web_enterprise` — the branding / theming layer

**What it does:** Replaces parts of Odoo's web UI with HCPI's look — navbar layout, SCSS variables, app launcher behaviour, sometimes the login screen.

**Why HCPI uses it:** the out-of-the-box Odoo UI is generic enterprise look. HCPI's branding (colours, logo placement, app tile ordering) sits in here.

**`kola_web_enterprise` vs `hcpi_brand`:**

| Module | Scope |
|---|---|
| `kola_web_enterprise` | Deeper UI changes — navbar layout, app launcher, SCSS variables. Often replaces Odoo files. |
| `hcpi_brand` | Country-agnostic HCPI defaults — logo, login template, transactional emails. |

When you tweak look-and-feel you'll usually touch one or both. See [Reference Data](reference-data.md#hcpi_brand-branding-emails-login) for `hcpi_brand`.

**Where to look to change something:**

| You want to… | Open |
|---|---|
| Change the navbar | `kola_web_enterprise/static/src/` — find the OWL/QWeb template that overrides Odoo's navbar |
| Change app launcher behaviour | `kola_web_enterprise/static/src/menu.js` (or similar) |
| Tweak global SCSS variables | `kola_web_enterprise/static/src/scss/` — primary/accent colour vars |
| Replace the login page | The XML view inheritance for `web.login` is here |

## `progress_bar_customization` — change a single colour

**What it does:** A CSS-only override that changes the progress-bar colour from Odoo's default yellow to orange. Cosmetic only — no models, no logic.

**Why it exists:** small visual tweaks like this are easier to keep in their own module than mixed into `kola_web_enterprise` — easier to revert, easier to skip on installs where it doesn't apply.

**Where to look to change something:** literally one file — `progress_bar_customization/static/src/css/`. Change the colour hex, restart Odoo (`-u progress_bar_customization`), refresh.

## `base_import_inherit` — generic import extensions

**What it does:** Extends Odoo's `base_import` (the CSV/Excel import wizard) with **validation hooks** and **batch-insertion optimisations** that HCPI imports use. Country-specific validation (Uganda location codes, etc.) layers on top via the per-country `*_base_import` modules — see [Country Overlays](country-overlay.md).

**The model:**

`base_import.import` (inherited, transient) — adds:

- `use_optimized_import` boolean — when on, switches to bulk-insert mode that skips per-row Python hooks.
- Pre-insertion validation that prevents duplicates (e.g. "this outlet code already exists").

**Where to look to change something:**

| You want to… | Open |
|---|---|
| Change duplicate-detection logic | `base_import_inherit/models/base_import.py` — search for `_validate_*` |
| Add a new hook for country overlays to plug into | Add an empty method here, override in `xx_base_import/` |
| Disable the optimised mode for a specific model | Override `use_optimized_import` per-model |
| Improve import error messages | Same file — error messages are `UserError("...")` raises |

## How the infrastructure modules fit together

```
                      ┌──────────────────┐
                      │   hcpi.conf      │
                      │   [queue_job]    │
                      └────────┬─────────┘
                               │ configures
                               ▼
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────────┐
│  queue_job       │◄───┤  hcpi_index      │───►│ web_progress         │
│  (runs jobs)     │    │  (dispatches     │    │ (shows progress      │
│                  │    │   compute jobs)  │    │  in the UI)          │
└──────────────────┘    └──────────────────┘    └──────────────────────┘

┌──────────────────┐    ┌──────────────────┐    ┌──────────────────────┐
│  base_import_    │◄───┤  ug_base_import  │    │ kola_web_enterprise  │
│  inherit         │    │  (Uganda-        │    │ + hcpi_brand         │
│  (generic        │    │   specific)      │    │ (branding)           │
│   validation     │    └──────────────────┘    └──────────────────────┘
│   hooks)         │
└──────────────────┘

┌──────────────────┐
│ progress_bar_    │
│ customization    │   (purely cosmetic — orange progress bar)
└──────────────────┘
```

## Common gotchas

- **Forgetting `server_wide_modules`.** If `queue_job` isn't loaded server-wide, jobs queue but never run. The fix is in `hcpi.conf`, not in code.
- **Touching vendored modules.** Patches here are easy to lose on the next vendor update. Prefer extending via `_inherit` in your own module.
- **Branding diffused across two modules.** When something looks off, check **both** `hcpi_brand` and `kola_web_enterprise` — they overlap.

## Next

You've now seen every module HCPI ships. Use the [Module Deep Dives index](index.md) and its quick-jump table to navigate back to any module by the change you want to make.

Or, if you came here looking for the broader picture:

- **[Module Reference](../understanding-the-codebase/module-reference.md)** — the bird's-eye view.
- **[Country Variants](../understanding-the-codebase/country-variants.md)** — branches, overlays, and country adoption.
- **[Configuration File](../training/day1/configuration.md)** — what every `hcpi.conf` option does, including `[queue_job]`.
