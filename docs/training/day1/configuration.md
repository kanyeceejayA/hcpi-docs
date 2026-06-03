# Configuration File

`hcpi.conf` is the file that tells Odoo where to find HCPI, how to connect to the database, what port to listen on, and a dozen other things. Understanding it is what separates "I followed the install script" from "I can run HCPI."

The configuration file lives at `/opt/hcpi/conf/hcpi.conf`. It ships pre-populated as part of `hcpi-files.zip` — you don't write it from scratch, you edit specific lines.

## How the file is structured

It's an INI file with sections in square brackets:

```ini
[options]
db_host = localhost
db_port = 5432
...

[queue_job]
channels = root:3,root.hcpi_ea:1
...
```

`[options]` is Odoo's main section — all the standard Odoo settings go here. `[queue_job]` is specific to the `queue_job` module HCPI uses for background tasks (heavy computations, bulk imports). Other modules can add their own sections.

To launch Odoo with this file:

```bash
python odoo/odoo-bin -c conf/hcpi.conf
```

The `-c` flag is what makes Odoo read it.

## The settings that matter on Day 1

There are 30+ settings in the file. Most are fine at their default. These are the ones you actually need to understand and possibly change:

### `addons_path` — the single most important line

```ini
addons_path = /opt/hcpi/odoo/addons,/opt/hcpi/custom/HCPI
```

This is a **comma-separated list of folders Odoo searches for modules**. Each folder in the list is expected to *contain* module folders (not *be* one).

So this setting says: "Look in `/opt/hcpi/odoo/addons/` (Odoo's built-in modules like `mail`, `web`, `base`) and in `/opt/hcpi/custom/HCPI/` (HCPI's own modules like `hcpi_outlet`, `hcpi_item`)."

**Three common ways people get this wrong:**

1. **Pointing at a module instead of a folder of modules.** `addons_path = /opt/hcpi/custom/HCPI/hcpi_outlet` is wrong — Odoo will try to find module folders *inside* `hcpi_outlet/` and find none.
2. **Forgetting the Odoo core addons.** Without `/opt/hcpi/odoo/addons`, Odoo can't load `base`, `web`, `mail`, etc. — and nothing works.
3. **Custom path mismatch.** If you installed to a path other than `/opt/hcpi`, you must update this. The exported file assumes `/opt/hcpi`.

You can add more folders to the list as you create new modules:

```ini
addons_path = /opt/hcpi/odoo/addons,/opt/hcpi/custom/HCPI,/opt/hcpi/custom/my_country
```

### Database connection: `db_host`, `db_port`, `db_user`, `db_password`

The exported file ships with:

```ini
db_host = False
db_port = False
db_user = hcpi
db_password = False
```

`db_host = False` tells Odoo to use the Unix socket and rely on **peer authentication** — matching the running Linux user's name against PostgreSQL's role names. This works on a production server where HCPI runs as a Linux user named `hcpi`.

In your training/development environment, you're running as yourself (not as a Linux user `hcpi`), so peer auth fails. You change these to:

```ini
db_host = localhost
db_port = 5432
db_user = hcpi
db_password = your_secure_password   ; the one you set in PostgreSQL Setup
```

`db_host = localhost` forces a TCP connection, which uses password auth.

The [PostgreSQL Setup](postgresql.md) page covers why this works the way it does.

### `db_name` — which database to use

```ini
db_name = hcpi
```

Pins HCPI to a specific database — without this, Odoo would show a database selector at first launch. The exported config sets this to `hcpi` (matching the database you created earlier).

### `http_port` — what to listen on

```ini
http_port = 9201
```

HCPI listens on 9201 by default. Change it if 9201 is in use on your machine:

```bash
sudo ss -ltnp | grep 9201        # find what's holding the port
```

Pick another free port (9202, 9203, ...) and update the config.

### `logfile` — where logs go

```ini
logfile = /opt/hcpi/log/hcpi.log
```

If you change the install path, update this. To tail logs while developing:

```bash
tail -f /opt/hcpi/log/hcpi.log
```

You'll spend a lot of time looking at this file.

### `admin_passwd` — the master password

```ini
admin_passwd = passwd
```

This is **not** a user login password. It's the password Odoo asks for when you do database-level operations through the web UI's database manager: create, duplicate, drop, backup, restore. It's a single shared secret per Odoo install.

- For training, leave it as it ships.
- For a real deployment, change it and store it somewhere safe (it's plain text on disk; protect the file).
- Lose it and you can no longer use the web-based DB manager — the only way back is to edit this file directly.

More detail on the [User Administration](user-administration.md) page.

### `server_wide_modules`

```ini
server_wide_modules = base,web,queue_job
```

Modules loaded into the Odoo process at startup, before any database is opened. `base` and `web` are core; `queue_job` is added by HCPI because the background job runner has to be available process-wide.

If you ever get a "module `queue_job` not found" error on startup, the cause is usually that `queue_job` isn't in `custom/HCPI/` (incomplete export). Either re-extract from your source server, or as a temporary workaround remove `queue_job` from this line.

### `proxy_mode`

```ini
proxy_mode = True
```

Tells Odoo to trust `X-Forwarded-*` headers — necessary when you're behind a reverse proxy (Apache/Nginx with HTTPS). In your training environment you're not behind a proxy, so it doesn't matter; leave it.

### Performance & resource limits

```ini
workers = 4
limit_memory_soft = 4147483608
limit_memory_hard = 4684354560
limit_time_cpu = 999999
limit_time_real = 999999
```

`workers > 0` switches Odoo into multi-process mode (one master + N worker processes). For training and small-scale dev, you can leave this — but on a 4 GB laptop, four workers may be more than you have headroom for. If startup is slow or your machine becomes sluggish:

```ini
workers = 0
```

`workers = 0` runs Odoo in single-threaded mode, which is fine for training and uses less RAM. Trade-off: no parallel request handling and the longpolling/queue_job background runner won't operate the same way — but you won't notice for Day 1 work.

`limit_time_cpu` and `limit_time_real` are absurdly high (999999) in the HCPI config on purpose: some HCPI computations and imports legitimately run for many minutes, and the default Odoo limits would kill them. Leave these alone unless you know why you're changing them.

## A worked diff: what trainees actually change

If you're following the WSL install, you'll change exactly these lines from the exported defaults:

```ini
db_host = localhost           ; was: False
db_port = 5432                ; was: False
db_password = your_secure_password  ; was: False
```

That's it. Everything else can stay as it ships, assuming you installed at `/opt/hcpi`. If you installed somewhere else, also update:

```ini
addons_path = /your/path/odoo/addons,/your/path/custom/HCPI
logfile = /your/path/log/hcpi.log
```

## Command-line overrides

Anything in the config file can be overridden on the command line. Useful flags for development:

| Flag | What it does |
|---|---|
| `-i HCPI` | Install the `HCPI` module (used once on first start with an empty DB) |
| `-u HCPI` | Update the `HCPI` module — reloads view XML, runs migrations |
| `-u all` | Update every installed module |
| `--dev=all` | Dev mode: live-reload of views, more verbose logs, no asset caching |
| `--log-level=debug` | More verbose logging than the config's `info` |
| `--stop-after-init` | Run init/update and exit cleanly — useful for scripted module installs |

Example: reload all views and assets after editing XML:

```bash
python odoo/odoo-bin -c conf/hcpi.conf -u all --dev=all
```

The next page ([User Administration](user-administration.md)) explains when you use `--dev=all` versus the asset regeneration UI.

## Practical steps

- **[Windows WSL](../../installation/windows-wsl.md)** — Step 10 (config edits) and Step 12 (first start with the right flags)
- **[Linux Server](../../installation/linux.md)** — Step 7

➡️ Next: [User Administration](user-administration.md)
