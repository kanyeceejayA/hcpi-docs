# Installing Dependencies

HCPI is mostly Python, but it needs a layer of system-level libraries underneath. This page walks through *what* each dependency is for — so when something fails to install you have a fighting chance of knowing why.

## The single command

The full install command from the [WSL guide](../../installation/windows-wsl.md):

```bash
sudo apt install -y git python3 python3-pip python3-dev python3-venv \
    postgresql postgresql-contrib libpq-dev \
    build-essential libssl-dev libffi-dev libxml2-dev libxslt1-dev \
    zlib1g-dev libjpeg-dev libsasl2-dev libldap2-dev \
    node-less npm wkhtmltopdf unzip
```

Twenty-odd packages. Below is what each one is for, grouped.

## Python toolchain

| Package | What it's for |
|---|---|
| `python3` | The Python interpreter HCPI runs on (3.10+) |
| `python3-pip` | Python's package installer — used inside the venv to install Odoo's deps |
| `python3-dev` | Python C headers — needed when pip has to compile a C extension (e.g. `lxml`, `psycopg2-binary` fallback) |
| `python3-venv` | The `venv` module — creates the isolated environment we install everything into |

## PostgreSQL

| Package | What it's for |
|---|---|
| `postgresql` | The database server itself |
| `postgresql-contrib` | Useful extensions bundled with PostgreSQL (we don't strictly need all of them, but it's the conventional install) |
| `libpq-dev` | C client library for PostgreSQL — `psycopg2` (the Python driver) compiles against this |

More on the database itself on the [PostgreSQL Setup](postgresql.md) page.

## Build tools and C libraries

Many of Odoo's Python dependencies have C components. If pre-built wheels aren't available for your platform, pip falls back to building from source — and that needs a C compiler plus a few common libraries.

| Package | What's compiled against it |
|---|---|
| `build-essential` | `gcc`, `make`, and friends — the compiler toolchain |
| `libssl-dev` | OpenSSL — used by Python's `cryptography` package |
| `libffi-dev` | Foreign Function Interface — also `cryptography` |
| `libxml2-dev`, `libxslt1-dev` | XML parsing — `lxml` (Odoo uses heavily for views, QWeb templates) |
| `zlib1g-dev` | Compression — used by Pillow, lxml, others |
| `libjpeg-dev` | JPEG support for Pillow (image processing) |
| `libsasl2-dev`, `libldap2-dev` | Authentication libraries — used if you ever wire Odoo into LDAP |

You won't see these mentioned again, but without them `pip install -r requirements.txt` will fail with a compile error somewhere deep in the trace.

## Frontend & reports

| Package | What it's for |
|---|---|
| `node-less` | LESS → CSS compiler — Odoo uses LESS for its stylesheets |
| `npm` | Node package manager — Odoo's web client tooling needs it for asset bundling |
| `wkhtmltopdf` | Headless WebKit-to-PDF converter — Odoo uses this to render reports to PDF |

`wkhtmltopdf` is the one most likely to cause trouble in production deployments because Odoo expects a specific patched version. The Ubuntu package version usually works for development, but for production you may need a hand-built binary. Not a Day 1 concern.

## Utilities

| Package | What it's for |
|---|---|
| `git` | Cloning Odoo 18 from GitHub |
| `unzip` | Extracting `hcpi-files.zip` |

## Validating the install

After `apt install` completes, sanity-check the key tools:

```bash
python3 --version          # Expect 3.10 or later
psql --version             # Expect 14+
git --version
wkhtmltopdf --version
```

If any are missing, re-run the `apt install` line and check the output for failed packages.

## What's *not* in the system install

Two things people expect to find here but aren't:

- **Odoo itself.** Odoo isn't an apt package — we clone it from GitHub later (covered on [Virtual Environment & HCPI Install](virtualenv-install.md)).
- **Odoo's Python dependencies** (`werkzeug`, `lxml`, `psycopg2`, `passlib`, ...). These go inside the Python venv via `pip install -r odoo/requirements.txt`. We install them *into the venv* deliberately — they don't go in the system Python.

The system packages above are just the foundation those next layers build on.

## Practical steps

Follow the install guide for your machine:

- **[Windows WSL](../../installation/windows-wsl.md)** — Step 4
- **[Linux Server](../../installation/linux.md)** — Step 2
- **[Windows Native](../../installation/windows-native.md)** — the dependencies page within

➡️ Next: [PostgreSQL Setup](postgresql.md)
