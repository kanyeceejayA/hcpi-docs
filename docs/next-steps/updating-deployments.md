# Updating a Country Deployment

Each country runs its own HCPI deployment off its own branch — Uganda on `ug_18`, Kenya on `ke_18`, Tanzania on `tz_18`, Zanzibar on `zar_18` (see [Country Variants](../understanding-the-codebase/country-variants.md) for the full map). "Updating" a deployment means pulling the latest commits for **that country's branch** from the canonical EAC repo and restarting the service so the new code loads.

This page is the routine for doing that safely. It assumes the code lives in `/opt/hcpi/custom/HCPI` and the config is `/opt/hcpi/conf/hcpi.conf` (the paths used throughout this documentation).

!!! info "One branch per deployment"
    A deployment's database is built against its branch's code. Always pull the **same branch** the deployment already runs — never switch a live deployment to a different country's branch, or the installed modules won't match the database.

## Step 1: Add the `eac` remote (first time only)

If this machine has never pushed to or pulled from the EAC repo, add it as a remote. Skip this step if `git remote -v` already lists `eac` (or an `origin` pointing at `East-African-Community-HCPI/HCPI`).

```bash
cd /opt/hcpi/custom/HCPI
git remote add eac git@github.com:East-African-Community-HCPI/HCPI.git
git fetch eac
```

!!! tip "Full remote setup lives on its own page"
    SSH-vs-HTTPS, key setup, access requests, and replacing `origin` are all covered in detail on [Git Remotes (EAC)](../getting-started/git-remotes.md). Come back here once `git fetch eac` works.

## Step 2: Make `eac` the default for this branch

Point the current branch's upstream at `eac` so plain `git pull` and `git push` target the EAC repo without you naming it every time:

```bash
# replace ug_18 with this deployment's branch
git branch --set-upstream-to=eac/ug_18
```

Optionally make `eac` the default remote for the whole repo, so new branches and pushes go there too:

```bash
git config remote.pushDefault eac
git config checkout.defaultRemote eac
```

## Step 3: Check the current branch

Confirm which branch this deployment is on **before** pulling — this is what tells you the country, and what you pass in the next step:

```bash
git branch --show-current      # e.g. ug_18
git status                     # should say "nothing to commit, working tree clean"
```

!!! warning "Uncommitted local changes?"
    If `git status` shows modified files, a previous edit was made directly on the server. Stash or commit it before pulling, or the pull will refuse to run:
    ```bash
    git stash            # set the changes aside
    # ... do the update ...
    git stash pop        # bring them back, then resolve any conflicts
    ```

## Step 4: Pull the latest for that branch

```bash
git fetch eac
git pull --ff-only eac ug_18   # use the branch from Step 3
```

`--ff-only` keeps the deployment's history a clean fast-forward — it refuses to create a surprise merge commit. If it fails, the working tree has diverged (see the warning above).

??? note "`git pull` reports conflicts or refuses to fast-forward"
    The deployment has commits that aren't on `eac` (someone edited on the server). Inspect what diverged with `git log --oneline eac/ug_18..HEAD`, then either push those commits to a branch for review or stash them. Don't force the pull — you'd lose server-side work.

## Step 5: Apply model or view changes (when needed)

Pure Python logic changes load on restart alone. But if the pulled commits touched **models/fields, views (XML), security, or data files**, you must upgrade the affected modules once so the database schema and views update:

```bash
cd /opt/hcpi
source venv/bin/activate
python odoo/odoo-bin -c conf/hcpi.conf -u hcpi_index --stop-after-init   # name the changed module(s), comma-separated
```

!!! tip "Not sure what changed?"
    `git diff --stat HEAD@{1} HEAD` shows the files the pull brought in. If any are under a module's `models/`, `views/`, `security/`, or `data/`, upgrade that module. When in doubt, `-u all` is safe but slower.

## Step 6: Restart

Restart so the running process picks up the new code. **Local development** is the default below; production servers running under systemd are in the collapsed section.

```bash
cd /opt/hcpi
source venv/bin/activate
python odoo/odoo-bin -c conf/hcpi.conf
```

Watch the log as it boots — `Registry loaded` with no `CRITICAL`/`Failed to load` lines means the update is live. Open the site and confirm.

??? note "On a production server (systemd)"
    A server deployment runs HCPI as the `hcpi` service (defined during [Linux installation](../installation/linux.md#running-as-a-service-recommended-for-production)), so you don't launch `odoo-bin` by hand — you restart the unit:

    ```bash
    sudo systemctl restart hcpi
    sudo systemctl status hcpi        # should show "active (running)"
    ```

    If you ran the `-u` upgrade in Step 5, do it through the service account or stop the service first so two processes don't touch the database at once:

    ```bash
    sudo systemctl stop hcpi
    sudo -u hcpi /opt/hcpi/venv/bin/python /opt/hcpi/odoo/odoo-bin \
        -c /opt/hcpi/conf/hcpi.conf -u hcpi_index --stop-after-init
    sudo systemctl start hcpi
    ```

    Follow the logs with `sudo journalctl -u hcpi -f` (or `tail -f` on the `logfile` set in the conf).

## Quick reference

```bash
cd /opt/hcpi/custom/HCPI
git remote add eac git@github.com:East-African-Community-HCPI/HCPI.git   # first time only
BR=$(git branch --show-current)                                          # this deployment's branch
git status                                                               # must be clean
git fetch eac && git pull --ff-only eac "$BR"

# apply + restart (local)
cd /opt/hcpi && source venv/bin/activate
python odoo/odoo-bin -c conf/hcpi.conf -u <changed_module> --stop-after-init
python odoo/odoo-bin -c conf/hcpi.conf

# apply + restart (server)
# sudo systemctl restart hcpi
```

| If you want to… | See |
|---|---|
| Set up SSH keys / push access to EAC | [Git Remotes (EAC)](../getting-started/git-remotes.md) |
| Confirm which branch a country runs | [Country Variants](../understanding-the-codebase/country-variants.md) |
| Diagnose a failed restart | [Troubleshooting](../troubleshooting.md) |
