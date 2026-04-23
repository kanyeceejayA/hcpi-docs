# Exporting HCPI from a Linux Server

This guide shows you how to extract an existing HCPI installation from a Linux server so you can clone it to another machine or country setup. You will end up with three files:

- `hcpi-files.zip` — the application code and configuration
- `hcpi.dump` — the database dump (PostgreSQL custom format)
- `hcpi-filestore.zip` — uploaded attachments and images

!!! info "Who this is for"
    If you can SSH into your country's server, you can follow these steps as-is. A handful of values (port, database name) may differ — you'll discover them in Step 1.

## Before you begin

You need:

- An SSH login for the server running HCPI (username, hostname/IP, port, password)
- `sudo` privileges on that server
- A Windows machine with PowerShell (built into Windows 10/11)

## Step 1: SSH into the server and find the installation

**What this step does:** Connects you to the HCPI server and shows you three things at once — which user owns HCPI, where it's installed, and where its config file lives.

Open PowerShell on your Windows machine and run:

```powershell
ssh youruser@your-hcpi-server -p <ssh_port>
```

Replace:

- `youruser` with your SSH username
- `your-hcpi-server` with the hostname or IP (e.g. `uboscpi.ubos.org`)
- `<ssh_port>` with the SSH port (often `22`; ask whoever set up the server)

Enter your password when prompted. Your prompt should change to something like `youruser@hcpi-server:~$`, meaning you're now on the server.

Now run:

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

Write these three values down — you'll need them in Step 3.

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

**What this step does:** Opens the HCPI configuration so you can capture the database name, ports, and passwords. These values drive every command in the rest of the guide.

The install folder is owned by the HCPI user, so you'll need `sudo` to read it:

```bash
sudo cat /opt/hcpi/conf/hcpi.conf
```

!!! tip "Use your path, not ours"
    `/opt/hcpi/conf/hcpi.conf` is the path on Uganda's server. If Step 1 showed a different path on your server, use that one instead.

From the output, find and note these values:

```ini
db_user     = hcpi          ; ← DB_USER
db_name     = hcpi          ; ← DB_NAME
db_host     = False         ; ← note if this is False or a real hostname
db_password = False         ; ← note if this is False or a real password
admin_passwd = UGhci876@    ; ← the master password (keep this safe)
http_port   = 9201          ; ← for reference
addons_path = /opt/hcpi/odoo/addons,/opt/hcpi/custom/HCPI
```

What these mean for you:

- **`db_host = False` and `db_password = False`**: HCPI connects to PostgreSQL through a local socket as the system user. You won't need a DB password to dump the database — Step 5 uses a trick that bypasses it.
- **`db_host = localhost` (or a hostname) with a real `db_password`**: HCPI uses a real password. Note it — Step 5 has an alternative command that uses it.
- **`admin_passwd`**: This is HCPI's master password (used in the database manager UI). Save it alongside the export — you'll want it when restoring elsewhere.

!!! warning "Treat these values as secrets"
    The `admin_passwd` and any `db_password` are sensitive. Don't paste them into chat messages or commit them to Git.

## Step 3: Set shell variables

**What this step does:** Saves the values you noted in Steps 1–2 into shell variables so every later command picks them up automatically. If your server uses different values, you only change them here.

If your Step 1/2 values differ from the examples (e.g. the install lives in a different folder or the DB is not named `hcpi`), edit the lines below before pasting.

If not, then copy and paste this whole block into your SSH session, then press Enter:

```bash
INSTALL=/opt/hcpi
RUN_USER=hcpi
DB_NAME=hcpi
DB_USER=hcpi
```



Verify the variables took effect:

```bash
echo "INSTALL=$INSTALL  DB_NAME=$DB_NAME  RUN_USER=$RUN_USER"
```

You should see the values you set echoed back.

!!! warning "These variables only last for the current SSH session"
    If you disconnect and reconnect, run Step 3 again before continuing.

## Step 4: Create a staging folder

**What this step does:** Creates a temporary folder on the server where we'll collect all the export files before downloading them.

```bash
STAGING=/tmp/hcpi-export
mkdir -p $STAGING
chmod 777 $STAGING
```

The `chmod 777` is important: in Step 5 the `postgres` system user needs to write into this folder, and it's a different user than you.

!!! info "Only want the code, not the data?"
    If your goal is a clean install with this country's HCPI modules but **no existing data**, you can skip Steps 5 and 7 (database and filestore). Just run Step 6 to get `hcpi-files.zip`, then follow your target machine's installation guide and pick "Start with Empty Instance" in the Restore step. Odoo will create a fresh database the first time it runs.

## Step 5: Export the database

**What this step does:** Produces a single compressed database dump file in PostgreSQL's "custom" format. It's smaller and faster to restore than a plain SQL file, and works across operating systems.

Dump the database. This works whether `db_password` was `False` or a real value — we run `pg_dump` as the `postgres` system user, which has full DB access via local socket authentication:

```bash
sudo -u postgres pg_dump -F c -f $STAGING/hcpi.dump $DB_NAME
```

Check the file was created:

```bash
ls -lh $STAGING/hcpi.dump
```

You should see a file sized anywhere from a few MB to several GB depending on data volume.

!!! info "If the command fails with an authentication error"
    Your server may have `pg_hba.conf` configured to require passwords even for the `postgres` user. Use this instead and enter the `db_password` from Step 2:

    ```bash
    pg_dump -U $DB_USER -h localhost -F c -f $STAGING/hcpi.dump $DB_NAME
    ```

## Step 6: Export the code and configuration

**What this step does:** Bundles the HCPI application code and config into a single zip matching the layout the installation guides expect.

Zip the `conf/` and `custom/` folders from the install. The `cd` has to happen **inside** the elevated shell — your own user can't enter `/opt/hcpi` (it's owned by the `hcpi` user), and `cd` is a shell builtin so plain `sudo cd` doesn't work. Running the `cd` + `zip` together under `sudo bash -c` handles both. The `cd` matters because it makes the zip store clean relative paths (`conf/`, `custom/`) instead of the full server path (`opt/hcpi/conf/...`), which would otherwise force whoever extracts it on Windows to dig through nested folders.

```bash
sudo bash -c "cd $INSTALL && zip -r $STAGING/hcpi-files.zip conf custom"
```

The `odoo/` folder is **not** included — it will be re-downloaded from GitHub on the target machine. The `venv/` and `log/` folders are also excluded.

Verify the zip has a clean top-level layout:

```bash
unzip -l $STAGING/hcpi-files.zip | head
```

You should see entries starting with `conf/` and `custom/` — **not** `opt/hcpi/conf/` or similar.

## Step 7: Export the filestore

**What this step does:** Packages the folder where Odoo stores uploaded attachments (images, PDFs, etc.). The database holds references to these files but not the files themselves — **without this step, attachments and images in your restored instance will be broken**.

### Find the filestore

The filestore lives under the run user's HOME directory — but on some servers HOME is `/home/hcpi`, on others it's `/opt/hcpi`, etc. Ask the system directly instead of guessing:

```bash
RUN_USER_HOME=$(getent passwd $RUN_USER | cut -d: -f6)
FILESTORE_PARENT=$RUN_USER_HOME/.local/share/Odoo/filestore
FILESTORE=$FILESTORE_PARENT/$DB_NAME

echo "Looking for filestore at: $FILESTORE"
sudo ls $FILESTORE
```

You should see a bunch of two-character folders (`01`, `02`, `03`, ... plus a `checklist` file). That's the filestore.

??? note "Didn't find it there?"
    The server may have overridden the default location. Try these in order:

    1. **Check the config for a `data_dir` line:**

        ```bash
        sudo grep -E '^\s*data_dir' /opt/hcpi/conf/hcpi.conf
        ```

        If it prints something, the filestore is at `<data_dir>/filestore/$DB_NAME`.

    2. **Search the filesystem:**

        ```bash
        sudo find / -type d -name filestore 2>/dev/null
        ```

        You may see several results (e.g. older Odoo installs). Pick the one that contains a subfolder matching your `$DB_NAME`. Then set the path manually:

        ```bash
        FILESTORE_PARENT=/the/path/you/found
        FILESTORE=$FILESTORE_PARENT/$DB_NAME
        ```

### Zip it

Once `sudo ls $FILESTORE` confirms the right folder, zip it. Same approach as Step 6 — `cd` into the filestore's parent inside the elevated shell, so the zip stores a clean top-level `$DB_NAME/` entry instead of the full server path:

```bash
sudo bash -c "cd $FILESTORE_PARENT && zip -r $STAGING/hcpi-filestore.zip $DB_NAME"
```

Check the result and confirm the layout:

```bash
ls -lh $STAGING/hcpi-filestore.zip
unzip -l $STAGING/hcpi-filestore.zip | head
```

The first entries should be `hcpi/` (or whatever your `$DB_NAME` is) and `hcpi/01/...` — **not** `home/hcpi/.local/share/Odoo/filestore/hcpi/...`.

## Step 8: Make the files readable by your SSH user

**What this step does:** The files were created by `sudo` and `postgres`, so they're owned by different users. This hands ownership to your SSH user so you can download them.

Replace `youruser` with your actual SSH username (the one you used in Step 1):

```bash
sudo chown $USER:$USER $STAGING/*
ls -lh $STAGING
```

You should see the database dump, files zip, and filestore zip all owned by you.

## Step 9: Copy the files to your Windows machine

**What this step does:** Downloads the three export files from the server to your local machine.

**Open a new PowerShell window on your Windows machine** (don't close the SSH session yet — you may want to clean up after). Create a local folder, then pull the files:

```powershell
mkdir C:\hcpi-export
scp -P <ssh_port> youruser@your-hcpi-server:/tmp/hcpi-export/* C:\hcpi-export\
```

Replace:

- `<ssh_port>` with your server's SSH port (often `22`, sometimes different — ask whoever set up the server)
- `youruser@your-hcpi-server` with your SSH login

You'll be prompted for your password. After the transfer, check the files arrived:

```powershell
dir C:\hcpi-export
```

You should see `hcpi.dump`, `hcpi-files.zip`, and `hcpi-filestore.zip`.

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
- **Odoo version** — confirm by reading the Odoo folder's git branch. The `odoo/` subdirectory is owned by the run user, so use `sudo`:

  ```bash
  sudo cat $INSTALL/odoo/.git/HEAD
  ```

  You should see something like `ref: refs/heads/18.0`. The number after `heads/` is the Odoo major version. HCPI installs are expected to run on `18.0` — flag it if you see anything else.

- **Python version** — run on the server:

  ```bash
  sudo $INSTALL/venv/bin/python3 --version
  ```

??? note "If `sudo cat .../HEAD` fails"
    As an alternative, use `git -C <path>` so git runs against the repo without needing to `cd` into it:

    ```bash
    sudo git -C $INSTALL/odoo log -1 --oneline
    ```

    You can then look up the commit on [github.com/odoo/odoo](https://github.com/odoo/odoo) to confirm which branch/version it belongs to.

## What next?

You now have everything needed to install HCPI somewhere else. Proceed to the installation guide that matches your target machine:

- [Linux Server Installation](../installation/linux.md)
- [Windows WSL Installation](../installation/windows-wsl.md)
- [Windows Native Installation](../installation/windows-native.md)

When those guides ask for `hcpi-files.zip`, use the one you just created. When they ask for the database dump, use `hcpi.dump` (the install guides include restore instructions for this format). The filestore zip can be extracted into the new machine's filestore folder after the first successful startup.

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
