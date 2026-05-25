# Virtual Environment & HCPI Install

Now the database is ready and the system has its libraries. Time to put Odoo and the HCPI code on disk, and to set up the Python environment they'll run in.

## What goes where

There are **three** pieces of code we're putting in place:

| Piece | Source | Lands at |
|---|---|---|
| Odoo 18 | `git clone` from GitHub | `/opt/hcpi/odoo/` |
| HCPI modules + config | `hcpi-files.zip` from your country's export | `/opt/hcpi/conf/` and `/opt/hcpi/custom/` |
| Python dependencies | `pip install -r odoo/requirements.txt` | `/opt/hcpi/venv/` |

The end state inside `/opt/hcpi`:

```
/opt/hcpi/
├── conf/      ← from hcpi-files.zip
├── custom/    ← from hcpi-files.zip
├── log/       ← created with mkdir
├── odoo/      ← cloned from GitHub
└── venv/      ← created with python3 -m venv
```

## Where the files come from

Three possible sources for the HCPI code — see [Getting the Code](../../getting-started/getting-the-code.md) for the full breakdown:

1. **Clone the EAC upstream repo** ([github.com/East-African-Community-HCPI/HCPI](https://github.com/East-African-Community-HCPI/HCPI), access from [mkakinyi@eachq.org](mailto:mkakinyi@eachq.org)) — canonical source, no data, starts empty. Right path for new deployments.
2. **Export from your country's server** via the [extraction guide](../../extraction/linux-export.md) — produces `hcpi-files.zip`, a database dump, and a filestore archive. Right path for migrating an existing instance.
3. **Uganda's published test files** ([statistics.ubos.org](https://statistics.ubos.org/shares/d/z_M6k4Jya_lxN6lWX5Wz_w)) — sample for evaluation only.

If you're going via option 2 or 3, the zip you end up with contains:

- **`conf/`** — `hcpi.conf` plus any other config the source server uses
- **`custom/HCPI/`** — every HCPI module (folders like `hcpi_outlet`, `hcpi_item`, etc.)

If you're cloning from the EAC repo, you only get the module folders — there's no `conf/`, so you'll create `hcpi.conf` yourself from the template on the [Getting the Code](../../getting-started/getting-the-code.md#create-the-config-file) page.

Odoo itself is **not** in any of the above. That's deliberate — Odoo is a 100+ MB framework that doesn't need to be packaged with HCPI. We pull it fresh from GitHub on each install:

```bash
git clone --depth 1 --branch 18.0 https://github.com/odoo/odoo.git
```

`--depth 1` skips history (we don't need it). `--branch 18.0` pins to Odoo 18, which is what HCPI is built against.

!!! warning "Use Odoo 18 specifically"
    HCPI modules use APIs and patterns specific to Odoo 18. If you accidentally clone `main` or another branch, things will break — sometimes loudly, sometimes subtly.

## Why a Python virtual environment?

A virtual environment ("venv") is an isolated Python installation that lives inside your project. When you install packages into it, they don't touch the system Python.

Why bother:

- **Versions can't conflict.** Odoo pins specific versions of `werkzeug`, `lxml`, `psycopg2`, etc. If you `pip install` those at the system level, you might overwrite versions some other program on your machine depends on (or some other Python app you install later might overwrite yours).
- **No `sudo` required.** Everything goes inside `/opt/hcpi/venv/`, which you own. No system-wide changes.
- **Easy reset.** If the venv gets into a weird state, delete it and recreate — your system Python is untouched.

Creating it is a one-liner:

```bash
cd /opt/hcpi
python3 -m venv venv
```

Activating it points `python` and `pip` at the venv's versions:

```bash
source venv/bin/activate
```

Your shell prompt will gain a `(venv)` prefix when it's active. From then on, `python` and `pip` refer to the venv's copies, and packages install into `/opt/hcpi/venv/lib/python3.x/site-packages/` rather than system-wide.

!!! tip "You need to re-activate every shell"
    Opening a new terminal? You're back outside the venv. Re-run `source /opt/hcpi/venv/bin/activate`. If you ever see a `ModuleNotFoundError` for something like `werkzeug` or `odoo`, the first thing to check is whether the venv is active.

To leave the venv: `deactivate`.

## Installing the Python dependencies

With the venv active:

```bash
pip install --upgrade pip
pip install wheel
pip install numpy
pip install -r odoo/requirements.txt
```

Line by line:

1. **Upgrade pip** — older pip versions occasionally fail on modern wheels. Quick safety move.
2. **Install `wheel`** — lets pip prefer pre-built wheels over source builds, which is much faster.
3. **Install `numpy` separately** — HCPI uses NumPy (for the validation algorithms and computation), but Odoo's `requirements.txt` doesn't include it. Installing it first avoids a surprise mid-deployment when an `import numpy` fails.
4. **Install Odoo's requirements** — the big one. This pulls in ~30 packages: `werkzeug`, `lxml`, `psycopg2-binary`, `passlib`, `Pillow`, `qrcode`, etc.

Expect 2–5 minutes the first time. On WSL, occasionally `pip` will fall back to compiling something from C (this is what `python3-dev` and the various `lib*-dev` packages from [Dependencies](dependencies.md) are for). If you get a compile error, the missing system package will be named in the trace — apt-install it and re-run.

## Why we install Odoo "as files" not via pip

You might notice we don't run `pip install odoo`. Odoo *is* a Python package, but for HCPI we use it as a source tree, not an installed package. Reasons:

- HCPI's `addons_path` points into `/opt/hcpi/odoo/addons` directly — easier than locating the installed package.
- It makes patching and reading Odoo's source trivial — open the folder in your IDE.
- The `odoo-bin` script lives at the root of the clone and is how we launch the server.

The `requirements.txt` we install gives the Odoo *source tree* everything *it* needs to run.

## Sanity check

After `pip install` finishes, with the venv active:

```bash
python -c "import odoo; print(odoo.__file__)"
```

You should see a path under `/opt/hcpi/odoo/odoo/__init__.py`. If it errors, the venv is wrong or the clone failed — check both.

## Practical steps

- **[Windows WSL](../../installation/windows-wsl.md)** — Steps 6, 7, 8 (directory creation, file transfer, venv & pip install)
- **[Linux Server](../../installation/linux.md)** — Steps 4, 5, 6

➡️ Next: [Development Environment](dev-environment.md)
