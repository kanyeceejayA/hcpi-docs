# Updating a Country Deployment

Each country runs its own HCPI deployment off its own branch — Uganda on `ug_18`, Kenya on `ke_18`, Tanzania on `tz_18`, Zanzibar on `zar_18` (see [Country Variants](../understanding-the-codebase/country-variants.md) for the full map). "Updating" a deployment means pulling the latest commits for **that country's branch** from the canonical EAC repo and restarting the service so the new code loads.

This page is the routine for doing that safely. Every command below uses two shell variables — the deployment folder and its config file — so the same commands work whether your install lives in `/opt/hcpi`, `/opt/cpi18`, or anywhere else.

!!! info "Set `$INSTALL` and `$CONF` at the top of your session"
    Before running any of the steps below, set these two variables. They persist for the rest of the shell session and every later command picks them up:

    ```bash
    INSTALL=/opt/hcpi               # the deployment root
    CONF=$INSTALL/conf/hcpi.conf    # the config file Odoo runs with
    ```

    Edit both lines if your install lives elsewhere (e.g. `INSTALL=/opt/cpi18` and `CONF=$INSTALL/conf/cpi18.conf`). Not sure? `ps auxf | grep odoo-bin` shows the running process's `-c` argument — the file path right after `-c` is `$CONF`, and the folder above `odoo/` is `$INSTALL`.

!!! info "One branch per deployment"
    A deployment's database is built against its branch's code. Always pull the **same branch** the deployment already runs — never switch a live deployment to a different country's branch, or the installed modules won't match the database.

## Step 1: SSH access to GitHub from WSL (first time only)

Most teams run these updates from WSL, and the single most common reason `git fetch` / `git pull` fails is that this WSL instance has no SSH key registered with the GitHub account that can read the EAC repo. **If you see `Permission denied (publickey)`, this is the step that fixes it.**

Quick check — if this prints your GitHub username, you can skip the rest of this step:

```bash
ssh -T git@github.com
# Hi <your-username>! You've successfully authenticated, ...
```

Otherwise, run the following **inside the WSL shell** (not in PowerShell or Git Bash on the Windows side):

```bash
# 1. Generate a key (press Enter through all prompts to accept the defaults)
ssh-keygen -t ed25519 -C "your-email@example.com"

# 2. Print the public key — copy the whole line that starts with ssh-ed25519
cat ~/.ssh/id_ed25519.pub
```

Open <https://github.com/settings/keys> → **New SSH key**, paste the public key, give it a descriptive title (e.g. *"WSL on KE server"*), and save.

Test the connection again:

```bash
ssh -T git@github.com
```

You should now see the `Hi <your-username>!` line. If you do, move on to Step 2.

!!! warning "WSL gotchas that cause `Permission denied`"
    - **WSL has its own `~/.ssh/`** — completely separate from `%USERPROFILE%\.ssh\` on the Windows side. A key that works in Git Bash will not be picked up here. Generate the key inside WSL.
    - **Don't put the key under `/mnt/c/...`.** NTFS permissions don't translate to POSIX, and `ssh` will refuse a private key whose file mode isn't `600`. Keep it in the WSL home directory (`~/.ssh/id_ed25519`).
    - **Every WSL distro / instance needs its own key.** A fresh `wsl --install` starts with no keys at all.
    - **Agent not running?** If a later `git push` says `Could not open a connection to your authentication agent`, start one and add the key:
      ```bash
      eval "$(ssh-agent -s)"
      ssh-add ~/.ssh/id_ed25519
      ```

??? note "Authenticated but still denied — `Permission to East-African-Community-HCPI/HCPI.git denied`"
    `ssh -T` worked but `git push` fails with the message above. That means GitHub recognises your key but your account hasn't been granted write access to the EAC organisation. Request access from [mkakinyi@eachq.org](mailto:mkakinyi@eachq.org).

For HTTPS-with-Personal-Access-Token (no SSH) and the non-WSL setup, see [Git Remotes (EAC)](../getting-started/git-remotes.md).

## Step 2: Add the `eac` remote (first time only)

If this machine has never pushed to or pulled from the EAC repo, add it as a remote. Skip this step if `git remote -v` already lists `eac` (or an `origin` pointing at `East-African-Community-HCPI/HCPI`).

```bash
cd $INSTALL/custom/HCPI
git remote add eac git@github.com:East-African-Community-HCPI/HCPI.git
git fetch eac
```

## Step 3: Check the current branch

Find out which branch this deployment runs on — every step below uses it. Capture it into `$BR` so you can paste the same commands without retyping the name:

```bash
BR=$(git branch --show-current)
echo "$BR"                     # e.g. ug_18
git status                     # should say "nothing to commit, working tree clean"
```

!!! warning "Uncommitted local changes?"
    If `git status` shows modified files, a previous edit was made directly on the server. Stash or commit it before pulling, or the pull will refuse to run:
    ```bash
    git stash --include-untracked         # set the changes aside
    ```
    To recover the changes and bring them back, run
     ```bash
    git stash pop        # bring them back, then resolve any conflicts
    ```

## Step 4: Point this branch at `eac` (first time only)

Now that you know the branch, set its upstream to `eac` so plain `git pull` and `git push` target the EAC repo without naming it every time. You only need this once per branch — skip it on subsequent updates.

```bash
git branch --set-upstream-to=eac/"$BR"
```

??? note "Optional: make `eac` the default for the whole repo"
    So new branches and pushes go to `eac` too:

    ```bash
    git config remote.pushDefault eac
    git config checkout.defaultRemote eac
    ```

## Step 5: Back up the database and filestore

Before pulling any code or applying any schema upgrades, take a snapshot of the live database and the Odoo filestore. If anything in the steps that follow misbehaves — a bad migration, a corrupted view, a module that won't load — this backup is what lets you revert in minutes instead of hours.

Run this as the Linux user that runs Odoo (often `hcpi` or `odoo`) — that user already has the right filesystem permissions on the filestore, and PostgreSQL's default local auth will accept it as the database owner without a password. If you are logged in as someone else, `sudo -iu hcpi` (or the right username) first.

### Discover the database and filestore paths

The same pattern as the export guide ([Find the filestore](../extraction/linux-export.md#find-the-filestore)) — read the conf, then derive the filestore from the run user's HOME via `getent` so the path is correct regardless of which user is currently logged in:

```bash
DB=$(grep      '^db_name' "$CONF" | awk -F'=' '{print $2}' | xargs)
DB_USER=$(grep '^db_user' "$CONF" | awk -F'=' '{print $2}' | xargs)

# By convention the Linux user that runs Odoo matches the db_user. If not, set RUN_USER manually.
RUN_USER=$DB_USER
RUN_USER_HOME=$(getent passwd "$RUN_USER" | cut -d: -f6)
FILESTORE_PARENT=$RUN_USER_HOME/.local/share/Odoo/filestore
FILESTORE=$FILESTORE_PARENT/$DB
```

Echo everything back so you can sanity-check that nothing is blank before you do anything destructive — a missing variable here will silently break the backup or, worse, write the dump to the wrong place:

```bash
cat <<EOF
INSTALL          = $INSTALL
CONF             = $CONF
DB               = $DB
DB_USER          = $DB_USER
RUN_USER         = $RUN_USER
RUN_USER_HOME    = $RUN_USER_HOME
FILESTORE_PARENT = $FILESTORE_PARENT
FILESTORE        = $FILESTORE
EOF
```

Every line should have a value to the right of `=`. If `DB`, `DB_USER`, or `RUN_USER_HOME` is empty, `$CONF` is probably wrong (re-check the path against `ps auxf | grep odoo-bin`) or the conf uses a different key name (e.g. `db_name` is commented out). Fix that before continuing.

Now confirm the filestore directory actually exists at the discovered path:

```bash
sudo ls "$FILESTORE" | head
```

You should see a bunch of two-character folders (`01`, `02`, `03`, …) plus a `checklist` file. That confirms the path is right.

??? note "Didn't find it there?"
    Same fallback ladder as the export guide:

    1. **Check the config for an explicit `data_dir`:**

        ```bash
        sudo grep -E '^\s*data_dir' "$CONF"
        ```

        If it prints something, override the discovered path with `<data_dir>/filestore`:

        ```bash
        FILESTORE_PARENT=<the data_dir>/filestore
        FILESTORE=$FILESTORE_PARENT/$DB
        ```

    2. **Search the filesystem:**

        ```bash
        sudo find / -type d -name filestore 2>/dev/null
        ```

        Several results are possible (old Odoo installs, multi-instance hosts). Pick the one that contains a subfolder matching your `$DB`, then set `FILESTORE_PARENT` to that and re-run the `sudo ls "$FILESTORE"` check.

    3. **Confirm the live process is using the conf you think it is:** `ps -ef | grep odoo-bin` and look at the `-c` argument. If it's a different conf, repeat Step 5 against that one.

### Take the backup

```bash
TS=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR=$INSTALL/backups
mkdir -p "$BACKUP_DIR"

# 1. Database dump (custom format — compressed, supports pg_restore)
pg_dump -U "$DB_USER" -Fc "$DB" -f "$BACKUP_DIR/${DB}-${TS}.dump"

# 2. Filestore (attachments and binary fields). -C makes the tar store a clean
#    "$DB/" top-level entry, the same trick the export guide uses with zip.
tar -czf "$BACKUP_DIR/filestore-${DB}-${TS}.tar.gz" -C "$FILESTORE_PARENT" "$DB"

# 3. Record the current git commit so you can return to it
git -C "$INSTALL/custom/HCPI" rev-parse HEAD > "$BACKUP_DIR/git-head-${TS}.txt"

ls -lh "$BACKUP_DIR" | tail -5
```

The three files you should now see — `<db>-<ts>.dump`, `filestore-<db>-<ts>.tar.gz`, and `git-head-<ts>.txt` — together let you reconstruct the deployment exactly as it was right now.

??? note "If `pg_dump` asks for a password or refuses to connect"
    `pg_dump -U "$DB_USER"` relies on PostgreSQL's local `peer` or `trust` auth — the usual setup for HCPI — so it just works when you're logged in as the Odoo user. If your installation uses password auth, or you can't switch to the Odoo user, pick one of these fallbacks:

    **Pass the password from the conf:**

    ```bash
    DB_PASS=$(grep '^db_password' "$CONF" | awk -F'=' '{print $2}' | xargs)
    PGPASSWORD="$DB_PASS" pg_dump -U "$DB_USER" -Fc "$DB" -f "$BACKUP_DIR/${DB}-${TS}.dump"
    ```

    **Dump as the `postgres` superuser** (works regardless of how `db_user` is configured, but needs sudo and leaves a root-owned dump file that you may need to `chown` afterwards):

    ```bash
    sudo -u postgres pg_dump -Fc "$DB" -f "$BACKUP_DIR/${DB}-${TS}.dump"
    sudo chown "$USER:$USER" "$BACKUP_DIR/${DB}-${TS}.dump"
    ```

!!! tip "Keep backups outside the deployment folder"
    `$INSTALL/backups` is fine for a quick rollback, but for anything you care about long-term, copy these three files off the server (rsync to a backup host, push to object storage, etc.). A disk failure on the deployment server takes the in-tree backups with it.

??? warning "If something goes wrong — rolling back"
    Open this section **only if a later step fails** (the pull conflicts, the `-u` upgrade errors out, or the service refuses to boot after restart). The rollback is destructive — it replaces the current database with the snapshot you just took, so any data entered after the backup will be lost.

    **Stop the service first** so nothing writes while you restore:

    ```bash
    sudo systemctl stop hcpi
    ```

    **Revert the database** to the snapshot. The drop/recreate pattern is the most reliable — `pg_restore` against an existing database can leave orphaned objects. Run as the Odoo user (same as Step 5):

    ```bash
    DB=$(grep      '^db_name' "$CONF" | awk -F'=' '{print $2}' | xargs)
    DB_USER=$(grep '^db_user' "$CONF" | awk -F'=' '{print $2}' | xargs)
    TS=<the timestamp from Step 5>
    BACKUP_DIR=$INSTALL/backups

    dropdb     -U "$DB_USER" "$DB"
    createdb   -U "$DB_USER" -O "$DB_USER" "$DB"
    pg_restore -U "$DB_USER" -d "$DB" "$BACKUP_DIR/${DB}-${TS}.dump"
    ```

    If `db_user` lacks `CREATEDB` (rare on HCPI deployments but possible), or you hit auth errors, do the same three commands as the postgres superuser instead:

    ```bash
    sudo -u postgres dropdb   "$DB"
    sudo -u postgres createdb "$DB" -O "$DB_USER"
    sudo -u postgres pg_restore -d "$DB" "$BACKUP_DIR/${DB}-${TS}.dump"
    ```

    **Revert the filestore.** Re-derive `FILESTORE_PARENT` the same way Step 5 did (so it's correct even if you're now in a fresh session):

    ```bash
    RUN_USER=$DB_USER
    RUN_USER_HOME=$(getent passwd "$RUN_USER" | cut -d: -f6)
    FILESTORE_PARENT=$RUN_USER_HOME/.local/share/Odoo/filestore

    rm -rf "$FILESTORE_PARENT/$DB"
    tar -xzf "$BACKUP_DIR/filestore-${DB}-${TS}.tar.gz" -C "$FILESTORE_PARENT"
    ```

    If Step 5's "Didn't find it there?" fallback was used, set `FILESTORE_PARENT` to the same override value here.

    **Revert the code** to the commit you recorded:

    ```bash
    cd $INSTALL/custom/HCPI
    PREV=$(cat "$BACKUP_DIR/git-head-${TS}.txt")
    git reset --hard "$PREV"
    ```

    Alternatively, if you have not done anything else with git since the failed pull, `git reset --hard HEAD@{1}` jumps back to the pre-pull state using the reflog.

    **Bring the service back up** and confirm it boots cleanly:

    ```bash
    sudo systemctl start hcpi
    sudo systemctl status hcpi
    sudo journalctl -u hcpi -n 100      # check for CRITICAL / Failed to load lines
    ```

    Once you're stable, investigate the failure *before* attempting the update again — running the same broken pull a second time will produce the same result.

## Step 6: Pull the latest for that branch

If you `sudo -iu` switched users for the backup, your shell may not be inside the repo any more. `cd` back in before fetching:

```bash
cd "$INSTALL/custom/HCPI"
git fetch eac
git pull --ff-only eac "$BR"
```

`--ff-only` keeps the deployment's history a clean fast-forward — it refuses to create a surprise merge commit. If it fails, the working tree has diverged (see the warning above).

??? note "`git pull` reports conflicts or refuses to fast-forward"
    The deployment has commits that aren't on `eac` (someone edited on the server). Inspect what diverged with `git log --oneline eac/"$BR"..HEAD`, then either push those commits to a branch for review or stash them. Don't force the pull — you'd lose server-side work.

## Step 7: Apply model or view changes (when needed)

Pure Python logic changes load on restart alone. But if the pulled commits touched **models/fields, views (XML), security, or data files**, you must upgrade the affected modules once so the database schema and views update:

```bash
source "$INSTALL/venv/bin/activate"
python "$INSTALL/odoo/odoo-bin" -c "$CONF" -u hcpi_index --stop-after-init   # name the changed module(s), comma-separated
```

!!! tip "Not sure what changed?"
    `git diff --stat HEAD@{1} HEAD` shows the files the pull brought in. If any are under a module's `models/`, `views/`, `security/`, or `data/`, upgrade that module. When in doubt, `-u all` is safe but slower.

## Step 8: Restart

Restart so the running process picks up the new code. **Local development** is the default below; production servers running under systemd are in the collapsed section.

```bash
source "$INSTALL/venv/bin/activate"
python "$INSTALL/odoo/odoo-bin" -c "$CONF"
```

Watch the log as it boots — `Registry loaded` with no `CRITICAL`/`Failed to load` lines means the update is live. Open the site and confirm.

??? note "On a production server (systemd)"
    A server deployment runs HCPI under a systemd unit (defined during [Linux installation](../installation/linux.md#running-as-a-service-recommended-for-production)), so you don't launch `odoo-bin` by hand — you restart the unit. **The unit name varies by deployment** — Uganda's is `hcpi`, others have shipped as `odoo`, `cpi18`, `eac-cpi`, etc. Set `SERVICE` once and reuse:

    ```bash
    SERVICE=hcpi                          # edit to match this deployment's unit name
    sudo systemctl restart "$SERVICE"
    sudo systemctl status  "$SERVICE"     # should show "active (running)"
    ```

    If you ran the `-u` upgrade in Step 7, do it through the service account or stop the service first so two processes don't touch the database at once. `RUN_USER` is whichever Linux user the unit runs as — usually the same as `db_user` from `$CONF`:

    ```bash
    RUN_USER=$(grep '^db_user' "$CONF" | awk -F'=' '{print $2}' | xargs)
    sudo systemctl stop "$SERVICE"
    sudo -u "$RUN_USER" "$INSTALL/venv/bin/python" "$INSTALL/odoo/odoo-bin" \
        -c "$CONF" -u hcpi_index --stop-after-init
    sudo systemctl start "$SERVICE"
    ```

    Follow the logs with `sudo journalctl -u "$SERVICE" -f` (or `tail -f` on the `logfile` set in the conf).

    ??? note "`Unit <name>.service could not be found` — how to discover the real name"
        If `systemctl status` reports the unit isn't loaded, the deployment was either registered under a different name or never set up as a systemd service at all. Find out which:

        **1. Ask the kernel which unit owns the running Odoo process** — the most reliable signal:

        ```bash
        PID=$(pgrep -fo "odoo-bin -c $CONF")
        cat /proc/$PID/cgroup
        ```

        A line like `0::/system.slice/<name>.service` is your unit — use that name. Anything under `user.slice/...` means the process was started by a login shell (nohup / screen / tmux), not systemd, and there is no unit to manage.

        **2. Grep unit files for this install's paths:**

        ```bash
        sudo grep -rEl "$INSTALL|$(basename "$CONF")" \
            /etc/systemd/system/ /lib/systemd/system/ 2>/dev/null
        ```

        The matching `.service` filename (minus the extension) is the unit name.

        **3. Sweep all registered units:**

        ```bash
        systemctl list-units      --type=service --all | grep -iE 'cpi|odoo|hcpi'
        systemctl list-unit-files --type=service       | grep -iE 'cpi|odoo|hcpi'
        ```

        **4. If none of the above match**, the install was started manually. Stop and restart it the same way:

        ```bash
        sudo -u "$RUN_USER" kill "$PID"
        sudo -u "$RUN_USER" nohup "$INSTALL/venv/bin/python" "$INSTALL/odoo/odoo-bin" \
            -c "$CONF" >/dev/null 2>&1 &
        ```

        Once you know the real name, write it down — it should become a registered systemd unit so future updates don't go through this lookup. The [Linux installation guide's service section](../installation/linux.md#running-as-a-service-recommended-for-production) covers how to set one up.

## Quick reference

```bash
# Set once per session — edit if your install lives elsewhere
INSTALL=/opt/hcpi
CONF=$INSTALL/conf/hcpi.conf

# first time only: SSH key + remote + upstream
ssh -T git@github.com                            # must say "Hi <your-username>!" before continuing
# if not, generate a key inside WSL and add it at https://github.com/settings/keys
#   ssh-keygen -t ed25519 -C "your-email@example.com"
#   cat ~/.ssh/id_ed25519.pub

cd "$INSTALL/custom/HCPI"
git remote add eac git@github.com:East-African-Community-HCPI/HCPI.git
BR=$(git branch --show-current)                  # this deployment's branch, e.g. ug_18
git branch --set-upstream-to=eac/"$BR"

# every update from here on
BR=$(git branch --show-current)
git status                                       # must be clean

# back up before pulling (see Step 5 for full commands, filestore discovery, and fallbacks)
DB=$(grep      '^db_name' "$CONF" | awk -F'=' '{print $2}' | xargs)
DB_USER=$(grep '^db_user' "$CONF" | awk -F'=' '{print $2}' | xargs)
RUN_USER_HOME=$(getent passwd "$DB_USER" | cut -d: -f6)
FILESTORE_PARENT=$RUN_USER_HOME/.local/share/Odoo/filestore
TS=$(date +%Y%m%d-%H%M%S)
mkdir -p "$INSTALL/backups"
pg_dump -U "$DB_USER" -Fc "$DB" -f "$INSTALL/backups/${DB}-${TS}.dump"
tar -czf "$INSTALL/backups/filestore-${DB}-${TS}.tar.gz" -C "$FILESTORE_PARENT" "$DB"
git rev-parse HEAD > "$INSTALL/backups/git-head-${TS}.txt"

git fetch eac && git pull --ff-only eac "$BR"

# apply + restart (local)
source "$INSTALL/venv/bin/activate"
python "$INSTALL/odoo/odoo-bin" -c "$CONF" -u <changed_module> --stop-after-init
python "$INSTALL/odoo/odoo-bin" -c "$CONF"

# apply + restart (server)
# SERVICE=hcpi                                   # see Step 8 systemd note for how to find your unit name
# sudo systemctl restart "$SERVICE"
```

| If you want to… | See |
|---|---|
| Set up SSH keys / push access to EAC | [Git Remotes (EAC)](../getting-started/git-remotes.md) |
| Confirm which branch a country runs | [Country Variants](../understanding-the-codebase/country-variants.md) |
| Diagnose a failed restart | [Troubleshooting](../troubleshooting.md) |
