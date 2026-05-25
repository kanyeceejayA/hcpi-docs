# Troubleshooting

The errors you're most likely to hit running or maintaining HCPI, with the fix that actually works. Search this page when something goes wrong — most issues fall into one of the categories below.

## Database / startup errors

### `psycopg2.OperationalError: ... Peer authentication failed for user "hcpi"`

The config still has `db_host = False` (or is missing `db_host` entirely). Odoo is using the Unix socket and PostgreSQL is trying peer auth — matching your Linux user to a PostgreSQL role of the same name. Since you're running as yourself (not as a Linux user `hcpi`), peer auth fails.

**Fix** in `/opt/hcpi/conf/hcpi.conf`:

```ini
db_host = localhost
db_port = 5432
db_password = <your_postgres_hcpi_password>
```

Background: [PostgreSQL Setup](training/day1/postgresql.md#authentication-peer-vs-password).

---

### `psycopg2.OperationalError: FATAL: password authentication failed for user "hcpi"`

`db_password` in `hcpi.conf` doesn't match what PostgreSQL has for the `hcpi` role.

**Fix:** reset the password to a known value:

```bash
sudo -u postgres psql -c "ALTER USER hcpi WITH PASSWORD 'your_secure_password';"
```

Then make sure the same value is in `hcpi.conf` as `db_password`.

---

### `could not connect to server: No such file or directory` / `Connection refused`

PostgreSQL isn't running.

**Fix:**

```bash
sudo systemctl start postgresql       # systemd-enabled systems
sudo service postgresql start         # WSL1 or older systems

# Confirm it's up
systemctl status postgresql
```

If it won't start: `sudo journalctl -u postgresql --no-pager -n 50` to see why.

---

### `database "hcpi" does not exist`

You haven't created the database yet, or you dropped it.

**Fix:**

```bash
sudo -u postgres createdb -O hcpi hcpi
```

If you want to populate it from a dump: see [PostgreSQL Setup](training/day1/postgresql.md#drop-and-recreate-the-database) for the restore flow.

---

### `psycopg2.errors.DuplicateObject: role "hcpi" already exists`

The PostgreSQL role exists from a previous install. Either reuse it (just make sure the password matches), or drop and recreate:

```bash
sudo -u postgres dropuser hcpi                                # only if no DBs depend on it
sudo -u postgres createuser -s hcpi
sudo -u postgres psql -c "ALTER USER hcpi WITH PASSWORD '...';"
```

---

### `pg_restore: error: relation "..." already exists`

You're re-running `pg_restore` against a database that already has tables in it. The dump expects an empty target.

**Fix:** drop and recreate the database, then re-run the restore.

```bash
# Stop Odoo first — it holds a connection that blocks dropdb
sudo -u postgres dropdb hcpi
sudo -u postgres createdb -O hcpi hcpi
pg_restore -U hcpi -h localhost -d hcpi --no-owner --no-privileges -j 4 hcpi.dump
```

If `dropdb` fails with "database is being accessed by other users", force-close connections first:

```bash
sudo -u postgres psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='hcpi' AND pid <> pg_backend_pid();"
sudo -u postgres dropdb hcpi
```

Also wipe the filestore if you're re-importing from scratch:

```bash
rm -rf ~/.local/share/Odoo/filestore/hcpi
```

---

## Module loading errors

### `ImportError: No module named '...'` / `ModuleNotFoundError`

Your venv isn't active, or a dependency didn't install.

**Fix:** activate the venv and reinstall:

```bash
cd /opt/hcpi
source venv/bin/activate
# Your prompt should now show (venv)

pip install --upgrade pip
pip install wheel
pip install numpy
pip install -r odoo/requirements.txt
```

If you don't see `(venv)` in your prompt after `source`, the path is wrong. The venv was created where you ran `python3 -m venv venv` — usually `/opt/hcpi/venv/`.

---

### `FileNotFoundError: ... HCPI` / module not found at startup

The `addons_path` in `hcpi.conf` is wrong. It must point at the **parent** folder of your modules, not at a module itself.

**Correct:**

```ini
addons_path = /opt/hcpi/odoo/addons,/opt/hcpi/custom/HCPI
```

This means "look in these two folders for module sub-folders".

**Wrong** (pointing at a single module):

```ini
addons_path = /opt/hcpi/custom/HCPI/hcpi_outlet
```

Update the path, restart. See [Configuration File](training/day1/configuration.md#addons_path-the-single-most-important-line).

---

### `queue_job` not found at startup

Two possibilities:

1. **The module isn't on disk.** Check `/opt/hcpi/custom/HCPI/queue_job/` exists. If not, the export was incomplete — re-extract from your source server.
2. **Temporary workaround:** remove `queue_job` from `server_wide_modules` in `hcpi.conf`:

    ```ini
    server_wide_modules = base,web
    ```

    Index computation will work synchronously instead of via background jobs.

---

### "Module already installed but cannot be found"

After deleting a module folder, the database still has metadata saying it's installed. Reset it:

```bash
sudo -u postgres psql -d hcpi -c "DELETE FROM ir_module_module WHERE name = 'the_module_name';"
```

Then restart and let `-u all` rebuild.

---

## Port and process errors

### `Port 9201 already in use`

Another process is on that port — usually a previous Odoo run that didn't exit cleanly, or a separate Odoo instance.

**Find it:**

```bash
sudo ss -ltnp | grep 9201
```

You'll see the PID. Either kill it (`sudo kill <pid>`) or change the port in `hcpi.conf`:

```ini
http_port = 9202
```

---

### Odoo hangs on startup, no error

Usually one of:

- **First start with large DB import.** Odoo builds asset bundles (JS/CSS) on first run; can take 1-3 minutes. Wait for `HTTP service (werkzeug) running on ... port 9201`.
- **`workers > 0` on a memory-constrained machine.** Workers each fork the full process; on a 4 GB laptop this can swap. Set `workers = 0` in `hcpi.conf` for development.
- **Stuck queue_job worker.** If you're running `--workers` mode and `queue_job` is initialized but unresponsive, the master may hang waiting. Set `workers = 0` to bypass.

---

## View / UI errors

### Changed a view file, no change visible in browser

The file change hasn't been applied to the database yet. Choose one:

**Option A: Restart with `-u`:**

```bash
python odoo/odoo-bin -c conf/hcpi.conf -u <module_name>
```

**Option B: Run with `--dev=xml` for live reload:**

```bash
python odoo/odoo-bin -c conf/hcpi.conf --dev=xml
```

Then refresh the browser (hard-refresh: `Ctrl+Shift+R`).

If the change still isn't visible after `-u`:

- **Wrong file?** Project-wide search for the exact string you changed.
- **Multiple matches?** Several modules may have inherited the same view. The most-derived one wins.
- **Browser cache.** Hard-refresh again, or open DevTools → Network → disable cache.

See [Making Your First Edits](first-edits/index.md) for the full edit → upgrade → refresh loop.

---

### Missing icons in the UI

The asset bundles got stale — common after a fresh install or a partial DB restore.

**Fix (UI):**

1. Top-right user menu → **Settings**
2. Activate developer mode if not already
3. **Settings → Technical → User Interface → Regenerate Assets Bundles**
4. Hard-refresh the browser

**Fix (CLI):**

```bash
python odoo/odoo-bin -c conf/hcpi.conf -u all --dev=all
```

`-u all` re-runs every module's install; `--dev=all` regenerates assets.

See [Windows WSL Installation](installation/windows-wsl.md#fix-missing-icons).

---

### "Access denied" / record invisible to a user

The user's group doesn't have permission on the model.

**Diagnose:**

1. With developer mode on, log in as admin
2. Go to **Settings → Users & Companies → Users**, open the user
3. Check the **Access Rights** tab — which groups are they in?
4. Cross-reference `security/ir.model.access.csv` in the relevant module — does any of their groups have read on the target model?

If not, either add them to a group that does, or add a new access line in the module's CSV. See Day 7 of the training for the full security model.

---

### "Internal Server Error" on a page

Read the Odoo log:

```bash
tail -100 /opt/hcpi/log/hcpi.log
```

The traceback at the bottom tells you the exception type and file. Common causes:

- **`AttributeError: 'NoneType' object has no attribute '...'`** — code expects a record but got an empty recordset. Often a `search()` returning nothing.
- **`KeyError`** — a field name doesn't exist on the model. Often a typo or a missing module.
- **`ValueError: ... constraint violation`** — DB constraint failed (e.g. NOT NULL on a missing field).
- **`AccessError`** — the user can't read or write the record (see "Access denied" above).

---

## Authentication / user errors

### Can't log in as admin

If you remember the password, just typing it should work. If you've forgotten it:

**With Odoo stopped**, set the password directly in the DB:

```bash
sudo -u postgres psql -d hcpi -c "UPDATE res_users SET password = 'admin' WHERE login = 'admin';"
```

Restart Odoo, log in as `admin`/`admin`, change the password from My Profile.

Full details: [User Administration](training/day1/user-administration.md#resetting-the-admin-password-youre-locked-out).

---

### Web DB manager rejects the master password

You're using `admin_passwd` from the wrong copy of `hcpi.conf`, or the file was edited and not saved.

**Diagnose:**

```bash
grep ^admin_passwd /opt/hcpi/conf/hcpi.conf
```

If it shows what you're typing in, restart Odoo (the value is read at startup). If the file shows a different value, that's the actual master password.

To change it: stop Odoo, edit `hcpi.conf`, restart.

---

### Forgot the master password, no plaintext in config

Modern Odoo stores `admin_passwd` as a hash after first use. If you've lost the original:

1. Stop Odoo
2. Edit `hcpi.conf`, replace the hash with a new plaintext value
3. Start Odoo — it'll re-hash on next use

---

## Index computation errors

### `Zero-price validation failed — insufficient safe months`

The `hcpi_computation` mixin requires at least 6 months of safe (non-zero) price history before allowing index computation. Less than that, and indices won't compute.

**Fix:** wait for more data, or — if this is a fresh deployment without history — temporarily relax the threshold in `hcpi_computation/models/hcpi_computation.py`. (Don't ship this change to production.)

---

### Index wizard finishes immediately, no data

The wizard dispatched a queue_job that failed silently. Check:

1. **Queue jobs view** — in HCPI: **Settings → Technical → Queue → Jobs**. Find the recent failed jobs and read their error messages.
2. **Log** — `tail /opt/hcpi/log/hcpi.log` while triggering the wizard. Errors in workers also show here.
3. **`server_wide_modules`** — confirm `queue_job` is listed. Without it, jobs don't run.

---

### Dashboard charts blank

In order of likelihood:

1. **No computed indices yet.** Trigger the index wizards (Settings → HCPI → trigger relevant index update) and wait for them to complete.
2. **`dashboard_display = False` on divisions.** The dashboard filters to divisions with `dashboard_display=True`. With developer mode on, open a `hcpi.class` record and toggle the flag.
3. **JavaScript error.** Open browser DevTools → Console. If `hcpi_dashboard.js` failed, you'll see the traceback. Often a stale asset bundle — regenerate (see "Missing icons" above).
4. **ApexCharts CDN unreachable.** The dashboard loads ApexCharts from a CDN. If the machine has no internet, charts won't render. Vendor ApexCharts locally as a fallback.

---

## File / filestore errors

### Attachments and images broken after restore

You restored the database but not the filestore. Odoo stores file content on disk, not in the DB.

**Fix:** unzip the filestore archive to the right location:

```bash
mkdir -p ~/.local/share/Odoo/filestore
cd ~/.local/share/Odoo/filestore
unzip /path/to/hcpi-filestore.zip
ls    # should show: hcpi   (or your db_name)
```

The folder name must match `db_name` in `hcpi.conf`. See [WSL Installation Step 11](installation/windows-wsl.md#a2-restore-the-filestore).

---

### `Permission denied` on `/opt/hcpi/log/hcpi.log` or filestore

You ran something as `root` previously (or as another user), and that file is now owned by the wrong user.

**Fix:**

```bash
sudo chown -R $USER:$USER /opt/hcpi
sudo chown -R $USER:$USER ~/.local/share/Odoo
```

---

## Performance issues

### Slow startup

- **Workers fork the master process** — `workers > 0` on low-RAM machines is slower than `workers = 0`. Try `workers = 0`.
- **First start is always slow** (asset bundling). Subsequent restarts should be < 30 seconds.
- **Big DB.** Indexes on `hcpi.outlet.item.observation` and `hcpi.outlet.item` make this manageable; if you're seeing slow startup with a big DB, check those indexes still exist (`\d hcpi_outlet_item_observation` in psql).

### Slow page loads

- **`workers = 0`** can't parallel-handle requests — fine for dev, painful with concurrent users.
- **Missing browser cache headers** — confirm `proxy_mode = True` if behind a reverse proxy.
- **Asset bundles regenerating on every request** — happens if `--dev=all` is on. Turn it off in production.

### Index computation slow

- Configure more queue_job workers in `hcpi.conf`:

    ```ini
    [queue_job]
    channels = root:8,root.hcpi_ea:2
    ```

    More channels = more parallelism. Be mindful of total worker count vs. available RAM.

- Check `pg_stat_activity` while computation runs — locking on observation tables is a common bottleneck at scale.

---

## When all else fails

1. **Re-read the bottom of the traceback.** Odoo errors are often clear once you isolate the actual exception line.
2. **Compare with a working install.** If a colleague has HCPI running, diff `hcpi.conf` and the module list.
3. **Wipe and reinstall.** For dev environments only — `dropdb hcpi`, `createdb -O hcpi hcpi`, restart with `-i HCPI`. Sometimes faster than diagnosis.
4. **Check the [Odoo 18 documentation](https://www.odoo.com/documentation/18.0/)** for framework-level errors.
5. **Ask in the Slack/team channel** with: (a) the exact command you ran, (b) the last 30 lines of the log, (c) the value of any relevant config setting.

## Related pages

- [Configuration File](training/day1/configuration.md) — what every setting does
- [PostgreSQL Setup](training/day1/postgresql.md) — common DB issues
- [User Administration](training/day1/user-administration.md) — password and access issues
- [Making Your First Edits](first-edits/index.md) — the edit → upgrade → refresh loop
