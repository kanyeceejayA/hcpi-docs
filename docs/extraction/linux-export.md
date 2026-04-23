# Exporting HCPI from a Linux Server

This guide shows you how to extract an existing HCPI installation from a Linux server so you can clone it to another machine or country setup. You will end up with three files:

- `hcpi-files.zip` — the application code and configuration
- `hcpi-db.zip` — the database dump
- `hcpi-filestore.zip` — uploaded attachments and images

!!! info "Who this is for"
    HCPI deployments across EAC countries were set up by the same company and share the same layout. If you can SSH into your country's server, you can almost certainly follow these steps as-is. Only a handful of values (port, database name) may differ — you'll discover them in Step 1.

## Before you begin

You need:

- SSH access to the server running HCPI (e.g. `ssh youruser@your-hcpi-server`)
- `sudo` privileges on that server
- An SSH client on your Windows machine (built into Windows 10/11 — just open PowerShell)

## Step 1: Find the installation

Log in over SSH, then run:

```bash
ps auxf | grep -E 'odoo-bin|hcpi'
```

You are looking for a line that looks like this:

```
hcpi  2119675  ... /opt/hcpi/venv/bin/python3 /opt/hcpi/odoo/odoo-bin -c /opt/hcpi/conf/hcpi.conf
```

That one line tells you everything:

| What you see | What it means |
|---|---|
| First column (`hcpi`) | The **user** that owns the installation |
| `/opt/hcpi/...` | The **install folder** |
| `-c /opt/hcpi/conf/hcpi.conf` | The **config file** path |

??? note "Didn't see anything? Try these fallbacks"
    If `ps` returned nothing, HCPI may not be running. Try:

    ```bash
    # List services that look HCPI-related
    systemctl list-units --type=service | grep -iE 'odoo|hcpi'

    # Or work backwards from the public URL via the reverse proxy
    ls /etc/apache2/sites-enabled/ 2>/dev/null
    ls /etc/nginx/sites-enabled/ 2>/dev/null
    ```

    Read the config it points to, find the backend port (e.g. `9201`), then:

    ```bash
    sudo ss -ltnp | grep 9201
    ```

    That will show the process and you can follow the `/proc/<PID>/cwd` symlink to the install folder.

## Step 2: Read the config file

The install folder is owned by the `hcpi` user, so you'll need `sudo` to read it:

```bash
sudo cat /opt/hcpi/conf/hcpi.conf
```

Find and note these values — you'll use them in later steps:

```ini
db_user = hcpi          ; ← DB_USER
db_name = hcpi          ; ← DB_NAME
http_port = 9201        ; ← HTTP_PORT (for reference)
addons_path = /opt/hcpi/odoo/addons,/opt/hcpi/custom/HCPI
```

!!! tip "Write these down"
    On Uganda's server these are all `hcpi`. On your server they might match, or the DB name might be different (e.g. `kenya_hcpi`). Whatever you see, note them down.

## Step 3: Set shell variables

Now set four shell variables. **Every command in the rest of this guide uses them**, so if your values differ, you only change them here.

```bash
INSTALL=/opt/hcpi
RUN_USER=hcpi
DB_NAME=hcpi
DB_USER=hcpi
```

!!! warning "These variables only last for the current SSH session"
    If you disconnect and reconnect, run Step 3 again before continuing.

## Step 4: Create a staging folder

A temporary folder on the server to collect the export files:

```bash
STAGING=/tmp/hcpi-export
mkdir -p $STAGING
```

## Step 5: Export the database

Dump the database using the `postgres` system user (no password needed — this works because HCPI uses local socket authentication):

```bash
sudo -u postgres pg_dump -f $STAGING/hcpi.sql $DB_NAME
```

Then zip it to match the filename used elsewhere in HCPI docs:

```bash
cd $STAGING
zip hcpi-db.zip hcpi.sql
```

Check the file was created and has a reasonable size:

```bash
ls -lh $STAGING/hcpi-db.zip
```

!!! info "If that fails with an authentication error"
    Your server may require a password. Use this instead, and enter the `db_password` from the config when prompted:

    ```bash
    pg_dump -U $DB_USER -h localhost -f $STAGING/hcpi.sql $DB_NAME
    ```

## Step 6: Export the code and configuration

Zip the `conf/` and `custom/` folders from the install:

```bash
cd $INSTALL
sudo zip -r $STAGING/hcpi-files.zip conf custom
```

The `odoo/` folder is **not** included — it will be re-downloaded from GitHub on the target machine. The `venv/` and `log/` folders are also excluded.

??? note "Excluding stray files"
    You may have old notes or log files mixed into `custom/HCPI/`. To skip them:

    ```bash
    sudo zip -r $STAGING/hcpi-files.zip conf custom \
      -x 'custom/HCPI/*.md' 'custom/HCPI/*.yaml'
    ```

## Step 7: Export the filestore

The filestore holds uploaded attachments (images, PDFs, etc.). It lives in the run user's home folder:

```bash
FILESTORE=/home/$RUN_USER/.local/share/Odoo/filestore/$DB_NAME
sudo ls $FILESTORE
```

If that lists files, zip it:

```bash
cd /home/$RUN_USER/.local/share/Odoo/filestore
sudo zip -r $STAGING/hcpi-filestore.zip $DB_NAME
```

!!! info "No filestore folder?"
    Some servers store it elsewhere. Check the config for a `data_dir = ...` line. If there isn't one, search:

    ```bash
    sudo find / -type d -name filestore 2>/dev/null
    ```

## Step 8: Make the files readable by your SSH user

The zips were created with `sudo`, so they are owned by `root`. Change ownership so your SSH user can download them. Replace `youruser` with your actual SSH username:

```bash
sudo chown youruser:youruser $STAGING/*.zip
ls -lh $STAGING
```

You should see all three zips owned by you.

## Step 9: Copy the files to your Windows machine

**Open a new PowerShell window on your Windows machine** (don't close the SSH session yet — you may want to clean up after). Create a local folder, then pull the files:

```powershell
mkdir C:\hcpi-export
scp -P <ssh_port> youruser@your-hcpi-server:/tmp/hcpi-export/*.zip C:\hcpi-export\
```

Replace:

- `<ssh_port>` with your server's SSH port (often `22`, sometimes different — ask whoever set up the server)
- `youruser@your-hcpi-server` with your SSH login

You'll be prompted for your password. After the transfer, check the files arrived:

```powershell
dir C:\hcpi-export
```

## Step 10: Clean up the server

Back in the SSH session, delete the staging folder so the export files don't sit on the server:

```bash
rm -rf $STAGING
```

## Step 11: Record what you exported

Before you forget, create a small notes file next to your zips on Windows so you remember what came from where. Open a text editor and save `C:\hcpi-export\EXPORT_NOTES.md` with:

- Source server hostname
- Date of export
- Values from Step 3 (`INSTALL`, `RUN_USER`, `DB_NAME`, `DB_USER`, `HTTP_PORT`)
- Odoo version — get it with:

  ```bash
  sudo -u $RUN_USER bash -c "cd $INSTALL/odoo && git log -1 --oneline"
  ```

- Python version:

  ```bash
  $INSTALL/venv/bin/python3 --version
  ```

## What next?

You now have everything needed to install HCPI somewhere else. Proceed to the installation guide that matches your target machine:

- [Linux Server Installation](../installation/linux.md)
- [Windows WSL Installation](../installation/windows-wsl.md)
- [Windows Native Installation](../installation/windows-native.md)

When those guides ask for `hcpi-files.zip` and `hcpi-db.zip`, use the ones you just created. The filestore zip can be extracted into the new machine's filestore folder after the first successful startup.

## Troubleshooting

### "Permission denied" when listing `/opt/hcpi`

Expected. The install is owned by the `hcpi` user. Prefix commands with `sudo`, or switch user with `sudo -iu hcpi`.

### `pg_dump` says the database doesn't exist

Re-check Step 2 — the `db_name` in the config is authoritative. On some servers it's not `hcpi`.

### `scp` is very slow or fails partway

Run it again — `scp` does not resume, but repeating the command will re-copy from scratch. For large filestores, consider `rsync` instead if it's available:

```powershell
rsync -avP -e "ssh -p <ssh_port>" youruser@your-hcpi-server:/tmp/hcpi-export/ C:\hcpi-export\
```

### The admin password in `hcpi.conf` is in plain text

That's normal for Odoo. Before sharing the export with anyone, consider opening `hcpi.conf` in the zip and replacing `admin_passwd` with a placeholder. The target country should set their own password during installation anyway.
