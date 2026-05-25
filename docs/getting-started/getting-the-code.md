# Getting the Code

Before you can install HCPI, you need the HCPI code on disk. There are **three sources** to choose from. Pick the one that matches your situation.

## At a glance

| Source | What you get | Best for | Has data? |
|---|---|---|---|
| **[Clone the EAC upstream repo](#option-1-clone-the-eac-upstream-repo)** | Latest HCPI source code, no data | New deployments; new country adoption; getting the canonical code | ❌ Empty instance |
| **[Export from a country's HCPI server](#option-2-export-from-a-country-server)** | Code + database dump + filestore | Migrating an existing instance to a new machine | ✅ Full data |
| **[Uganda's published test files](#option-3-ugandas-test-files)** | Code + sample database + sample filestore | Evaluation, training, reference | ✅ Sample data |

If you're not sure: **Option 1** is the right starting point for almost everyone setting up a new HCPI deployment.

## Option 1: Clone the EAC upstream repo

This is the canonical source — the same repository every country deployment branches from. You get the latest code, but no data. Your HCPI will start empty and you'll create the master admin user and seed the database during first run.

### Access

The repo is private. **Request access from [mkakinyi@eachq.org](mailto:mkakinyi@eachq.org)** with your GitHub username. You'll be added to the organization as a read-only collaborator (or full collaborator if you'll be contributing changes).

Repo URL: **<https://github.com/East-African-Community-HCPI/HCPI>**

### Clone

Once access is granted, clone the **`18.0` branch** (the shared upstream — the right starting point for new deployments):

```bash
sudo mkdir -p /opt/hcpi/custom
sudo chown $USER:$USER /opt/hcpi /opt/hcpi/custom
cd /opt/hcpi/custom
git clone --branch 18.0 https://github.com/East-African-Community-HCPI/HCPI.git
```

This creates `/opt/hcpi/custom/HCPI/` containing all the HCPI modules.

!!! tip "Country branches"
    If you're maintaining an existing country deployment, clone the appropriate country branch instead — `ug_18`, `ke_18_mtnce`, `tz_18_mtnce`, or `zar_18`. See [Country Variants](../understanding-the-codebase/country-variants.md) for the full branch zoo.

    For a brand-new country, stay on `18.0` and add your country's overlay modules in a new branch off it.

### Create the config file

Cloning gives you only the modules — it does **not** include `conf/hcpi.conf`. You need to create one from scratch:

```bash
mkdir -p /opt/hcpi/conf /opt/hcpi/log
nano /opt/hcpi/conf/hcpi.conf
```

Paste this minimal config:

```ini
[options]
admin_passwd = change_this_master_password
db_host = localhost
db_port = 5432
db_user = hcpi
db_password = your_secure_password
db_name = hcpi
addons_path = /opt/hcpi/odoo/addons,/opt/hcpi/custom/HCPI
logfile = /opt/hcpi/log/hcpi.log
http_port = 9201
server_wide_modules = base,web,queue_job
without_demo = True
workers = 0
limit_time_cpu = 999999
limit_time_real = 999999

[queue_job]
channels = root:3,root.hcpi_ea:1
scheme = http
host = localhost
port = 9201
```

Then edit the three highlighted values:

- **`admin_passwd`** — the master password for database-level operations (backup, restore, drop via Odoo's DB manager). Change to something secret.
- **`db_password`** — match the PostgreSQL password you'll set when creating the `hcpi` role during the installation.
- (For Windows native installs, swap `/opt/hcpi` for `C:\hcpi` throughout.)

Everything else in the template is sensible default. See [Configuration File](../training/day1/configuration.md) for what each setting does.

### Layout you'll end up with

```
/opt/hcpi/
├── conf/
│   └── hcpi.conf        ← you created this
├── custom/
│   └── HCPI/            ← cloned from the EAC repo
│       ├── hcpi_coicop/
│       ├── hcpi_item/
│       ├── hcpi_outlet/
│       └── ...
├── log/                 ← you created this
├── odoo/                ← cloned separately during install (Odoo 18)
└── venv/                ← created later during install
```

### Continue with the installation

Now jump to your platform's install guide and **skip the "unzip hcpi-files.zip" step** — your files are already in place:

- [Windows WSL Installation](../installation/windows-wsl.md)
- [Linux Server Installation](../installation/linux.md)
- [Windows Native Installation](../installation/windows-native.md)

Then at first start, use **Option B (empty database)** in the data setup step — there's no dump to restore.

## Option 2: Export from a country server

Use this if you're **migrating an existing HCPI instance** to a new machine, or cloning a running deployment for development.

You'll produce three artifacts from the source server:

- **`hcpi-files.zip`** — `conf/` + `custom/` folders (code + configuration)
- **`hcpi.dump`** — PostgreSQL custom-format database dump
- **`hcpi-filestore.zip`** — uploaded attachments and images

The [Exporting HCPI from a Linux Server](../extraction/linux-export.md) guide walks through producing all three.

You'll then transfer the three files to your destination machine and the install guide handles unpacking them. Both database and filestore must be restored together — restoring one without the other leaves broken attachments.

This option is **only useful if you have shell access to a running HCPI server**. If you're starting from scratch, use Option 1.

## Option 3: Uganda's test files

A zipped set of files from Uganda's instance, hosted publicly for **evaluation and training only**. Includes code, a sample database, and a sample filestore — so you can have a running HCPI in minutes without provisioning anything.

```bash
# From inside WSL or a Linux machine:
cd /opt/hcpi
wget https://statistics.ubos.org/shares/d/z_M6k4Jya_lxN6lWX5Wz_w/hcpi-files.zip
unzip hcpi-files.zip
```

The full set (including `hcpi.dump` and `hcpi-filestore.zip`) is at the same share URL: <https://statistics.ubos.org/shares/d/z_M6k4Jya_lxN6lWX5Wz_w>.

**Don't use this for production.** It's frozen at some snapshot point and won't include current upstream changes or your country's customizations.

## Which one is right for me?

| Your situation | Use |
|---|---|
| New country adopting HCPI | Option 1 (clone, empty DB) |
| Existing country, fresh dev machine | Option 1 (clone the country's branch) + restore latest DB dump |
| Migrating production from one machine to another | Option 2 (full export from source) |
| Joining the project and want a running instance to explore | Option 3 (Uganda test files) |
| Cloning another country's setup for reference | Option 2 (their export) or Option 1 (their branch) + empty DB |
| Setting up CI / a clean test environment | Option 1 (clone, empty DB) |

## What's next

Once you have the code on disk:

- **[Prerequisites](prerequisites.md)** — confirm your system meets the requirements
- **[Windows WSL Installation](../installation/windows-wsl.md)** — recommended for development on Windows
- **[Linux Server Installation](../installation/linux.md)** — for production
- **[Windows Native Installation](../installation/windows-native.md)** — for testing
