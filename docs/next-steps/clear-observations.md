# Clearing All Observation Lines

There are situations — usually data resets on a test instance, or a country office requesting a clean slate after a bad import — where every `hcpi.outlet.item.observation` record needs to be wiped. This page is the safe procedure for doing that.

!!! danger "This is destructive and irreversible"
    `hcpi.outlet.item.observation` is the **price observation** table — it holds every price quote ever collected. Once you run the script below, those records are gone. Computed indices that referenced them stay in the database but become statistically meaningless until new observations come in.

    **Do not run this on a production instance without an explicit written request** from the country office, a current backup, and a maintenance window agreed with the data team.

## When to use this

Typical reasons the procedure has been used historically:

- **Country office requested a reset** (Kenya, June 2026 — the originating request for this page).
- **Test or staging instance cleanup** before loading a fresh import.
- **Bad bulk import to revert**, where individual `unlink` from the UI is too slow.

If you are trying to fix a single bad batch or month, do **not** use this — filter to that subset using the validation views instead.

## What gets cleared, what stays

| Model | Cleared by this procedure? |
|---|---|
| `hcpi.outlet.item.observation` | **Yes** — every row deleted |
| `hcpi.data.collection.line` | No |
| `hcpi.data.observation.line` | No |
| `hcpi.data.collection` (questionnaires) | No |
| `hcpi.data.validation` (validation batches) | No |
| `hcpi.basket.*.index` and `hcpi.national.index` | No — stay in place, but reflect a now-empty observation table |

If the country office also wants questionnaires, validations, or computed indices reset, that is a separate conversation — extend the script with the relevant `env['<model>'].search([])` line per table, but only after explicitly agreeing the scope.

## Pre-flight

This procedure uses the same shell variables as [Updating a Country Deployment](updating-deployments.md). Set them once at the top of your session:

```bash
INSTALL=/opt/hcpi               # the deployment root
CONF=$INSTALL/conf/hcpi.conf    # the config file Odoo runs with
DB=$(grep '^db_name' "$CONF" | awk -F'=' '{print $2}' | xargs)
```

Edit `INSTALL` and `CONF` if your install lives elsewhere (e.g. `/opt/cpi18/conf/cpi18.conf`).

## Step 1: Back up the database and filestore

Take a full snapshot using the **exact same procedure as Step 5 of [Updating a Country Deployment](updating-deployments.md#step-5-back-up-the-database-and-filestore)**. The pre-clear backup is the only way to recover if the wrong instance, wrong database, or wrong table is hit.

Do not proceed to Step 2 until you have:

- `<db>-<ts>.dump` (the database dump)
- `filestore-<db>-<ts>.tar.gz` (the filestore tarball)
- Verified both files exist and have sensible sizes (`ls -lh "$INSTALL/backups"`)

## Step 2: Stop the running Odoo service

The clear runs from a separate Odoo shell process and writes directly to the database. Stopping the main service first guarantees no other process is reading or writing observations at the same time.

```bash
SERVICE=hcpi                          # edit to match this deployment's unit name
sudo systemctl stop "$SERVICE"
sudo systemctl status "$SERVICE"      # should show "inactive (dead)"
```

If `systemctl` says the unit isn't loaded, the deployment was either registered under a different name or never set up as a systemd service — see the [discovery procedure in updating-deployments.md](updating-deployments.md#step-8-restart) for how to find the real name (or stop a manually-started process).

## Step 3: Activate the venv and open an Odoo shell

The Odoo shell is a Python REPL with `env`, `self`, and the full ORM pre-loaded. It's the cleanest way to run multi-line ORM operations.

```bash
source "$INSTALL/venv/bin/activate"
python "$INSTALL/odoo/odoo-bin" shell -c "$CONF" -d "$DB"
```

The prompt becomes something like `>>>`. You are now inside a Python session with full access to the database.

??? note "If `shell` errors with `database is being accessed by other users`"
    Another process still has a connection. Confirm the main service is stopped (Step 2), then check for stragglers:

    ```bash
    sudo -u postgres psql -c "SELECT pid, usename, application_name, state \
        FROM pg_stat_activity WHERE datname = '$DB';"
    ```

    Kill any remaining sessions with `SELECT pg_terminate_backend(<pid>);` from a postgres-superuser psql, then retry.

## Step 4: Run the delete loop

Inside the shell, paste this block. It deletes in batches of 10 000 and commits after each batch so a crash mid-way doesn't roll back hours of work:

```python
Obs = env['hcpi.outlet.item.observation']
print(Obs.search_count([]))      # sanity-check the volume first

n = 0
while True:
    recs = Obs.search([], limit=10000)
    if not recs:
        break
    recs.unlink()
    env.cr.commit()              # commit each batch so progress is saved
    n += len(recs)
    print(n, "deleted")

print("done:", n)
```

What each line does:

| Line | Purpose |
|---|---|
| `Obs.search_count([])` | Prints the **total observation count** so you have a baseline to compare against — this is your last sanity check before deleting anything. |
| `Obs.search([], limit=10000)` | Pulls one batch at a time. The limit keeps memory bounded — `search([])` without a limit on a multi-million-row table will OOM the shell. |
| `recs.unlink()` | The actual delete — calls each model's `unlink` override so audit trails and onchange logic fire. |
| `env.cr.commit()` | Commits the batch immediately. Without this, the whole delete would happen in one transaction and a power cut at 90% would roll back everything. |
| `print(n, "deleted")` | Live progress to the terminal. On a large database this loop can run for many minutes — watch the counter rise. |

When the loop prints `done: <number>`, every observation row is gone. Exit the shell with `Ctrl-D` (or `exit()`).

!!! warning "Don't shrink the batch size silently"
    The batch limit of `10000` is a balance: large enough that the commit overhead is amortised, small enough that one rollback only loses a few seconds of work. Going below ~1 000 makes the procedure unnecessarily slow on multi-million-row tables; going above ~50 000 risks the shell's working set blowing out RAM.

## Step 5: Restart the service

```bash
sudo systemctl start "$SERVICE"
sudo systemctl status "$SERVICE"      # should show "active (running)"
sudo journalctl -u "$SERVICE" -n 50   # confirm a clean startup, no CRITICAL lines
```

Open the web UI and confirm the observation list views are empty as expected.

## Step 6: Notify the data team

Send the following to whoever requested the clear:

- The timestamp of the backup in Step 1 (and where it lives — `$INSTALL/backups/`).
- The pre-clear observation count printed by `search_count([])`.
- The post-clear count (run `env['hcpi.outlet.item.observation'].search_count([])` from a fresh shell — should be `0`).
- Confirmation that the service restarted cleanly.

## Rolling back

If the wrong instance was hit, or the clear was authorised in error, the only recovery is to restore from the Step 1 backup. Follow the **rollback procedure folded into Step 5 of [Updating a Country Deployment](updating-deployments.md#step-5-back-up-the-database-and-filestore)** — stop the service, drop and recreate the database, `pg_restore` the dump, restore the filestore tarball, restart.

There is no incremental undo; a `git revert`–style operation does not exist for the data. The backup is the only path back.

## Related procedures

- [Updating a Country Deployment](updating-deployments.md) — the backup commands and the service-stop / restart patterns reused here.
- [Troubleshooting](../troubleshooting.md) — common errors that surface after large data operations.
