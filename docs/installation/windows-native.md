# Windows Native Installation

This guide covers installing HCPI directly on Windows without WSL.

!!! info "Windows Native Installation"
    Windows native installation works well for development and testing. However, it's not recommended for production deployments. You may encounter a few oddities or minor bugs due to path handling differences and platform-specific behaviors. For production environments, use [Linux](linux.md). If you want the best Windows experience, [Windows WSL](windows-wsl.md) provides a Linux environment with better compatibility.

## Step 1: Enable PowerShell Script Execution

Before installing Python, enable PowerShell to run scripts. Open PowerShell as Administrator and run:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

This allows you to run local scripts, which is needed for some installation steps.

## Step 2: Install Python

!!! warning "Python Version Requirement"
    HCPI works best with Python 3.10, 3.11, or 3.12. Python 3.13+ may have compatibility issues with some Odoo dependencies.

### Check Current Python Version

First, check if you have Python installed:

```powershell
python --version
```

### If Python 3.13+ is Installed (or No Python)

<details>
<summary><b>Click here if you have Python 3.13+ or need to install Python 3.12</b></summary>

If you have Python 3.13 or later, use pyenv-win to install Python 3.12:

**Install pyenv-win:**
```powershell
Invoke-WebRequest -UseBasicParsing -Uri "https://raw.githubusercontent.com/pyenv-win/pyenv-win/master/pyenv-win/install-pyenv-win.ps1" -OutFile "./install-pyenv-win.ps1"
./install-pyenv-win.ps1
```

Close and reopen PowerShell, then install Python 3.12:

```powershell
pyenv install 3.12.8
pyenv global 3.12.8
```

Verify the version:
```powershell
python --version
```

</details>

### If Installing Python Fresh

Download and install Python 3.12.x from [python.org](https://www.python.org/downloads/):

1. Download the Python 3.12.x Windows installer
2. **Important**: Check "Add Python to PATH" during installation
3. Complete the installation

Verify installation:

```powershell
python --version
pip --version
```

## Step 3: Install PostgreSQL

Download and install PostgreSQL from [postgresql.org](https://www.postgresql.org/download/windows/):

1. Download the Windows installer (version 12 or later)
2. During installation:
   - Remember the password you set for the `postgres` user
   - Default port is 5432 (keep this unless you have conflicts)
   - Install pgAdmin 4 (helpful for database management)
3. Complete the installation

Add PostgreSQL to your PATH:
1. Open System Properties → Environment Variables
2. Edit `Path` variable
3. Add: `C:\Program Files\PostgreSQL\15\bin` (adjust version number as needed)

## Step 4: Install Git

Download and install Git from [git-scm.com](https://git-scm.com/download/win):

1. Download and run the installer
2. Use default options during installation

## Step 5: Install wkhtmltopdf

Download and install wkhtmltopdf from [wkhtmltopdf.org](https://wkhtmltopdf.org/downloads.html):

1. Download the Windows installer
2. Install to default location
3. Add to PATH: `C:\Program Files\wkhtmltopdf\bin`

## Step 6: Set Up PostgreSQL Database

You can do this with either `psql` on the command line or with pgAdmin 4 (installed alongside PostgreSQL in Step 3). Use whichever you're more comfortable with — pgAdmin 4 is a good fallback if `psql` isn't on your PATH or the CLI gives you trouble.

### Option A: Using psql (Command Line)

Open Command Prompt as Administrator and create user and database:

```cmd
psql -U postgres
```

Enter the postgres password you set during installation. Then in the PostgreSQL prompt:

```sql
CREATE USER hcpi WITH PASSWORD 'your_secure_password';
CREATE DATABASE hcpi OWNER hcpi;
GRANT ALL PRIVILEGES ON DATABASE hcpi TO hcpi;
\q
```

### Option B: Using pgAdmin 4 (GUI)

1. Open **pgAdmin 4** from the Start menu and connect to your local PostgreSQL server (enter the `postgres` password from Step 3 when prompted).
2. **Create the login role:**
    - In the browser tree, expand your server → right-click **Login/Group Roles** → **Create** → **Login/Group Role...**
    - **General** tab: Name → `hcpi`
    - **Definition** tab: Password → `your_secure_password`
    - **Privileges** tab: enable **Can login?** and **Create databases?**
    - Click **Save**
3. **Create the database:**
    - Right-click **Databases** → **Create** → **Database...**
    - **Database**: `hcpi`
    - **Owner**: `hcpi`
    - Click **Save**
4. (Optional) Grant all privileges — since `hcpi` is already the owner this is usually unnecessary, but if you want to match the CLI exactly, open the **Query Tool** on the `hcpi` database and run:

    ```sql
    GRANT ALL PRIVILEGES ON DATABASE hcpi TO hcpi;
    ```

!!! tip "Database User"
    We're using `hcpi` as both the database name and username. You can choose different names, but update the configuration file accordingly.

## Step 7: Create Directory Structure

Open PowerShell and create the directory:

```powershell
mkdir C:\hcpi
cd C:\hcpi
```

!!! warning "Custom Installation Path"
    If you choose a different path than `C:\hcpi`, you'll need to update the paths in the configuration file (see Step 11).

!!! tip "Alternative Location"
    While the Linux version uses `/opt/hcpi`, on Windows we use `C:\hcpi` for simplicity. You can choose a different location if preferred.

## Step 8: Set Up HCPI Files

By now you should already have `hcpi-files.zip` — produced from your country's server using the [extraction guide](../extraction/linux-export.md). See [Prerequisites → Get the Required Files](../getting-started/prerequisites.md#get-the-required-files) if you don't.

Extract `hcpi-files.zip` into `C:\hcpi`. You can right-click the zip → **Extract All...** → browse to `C:\hcpi` as the destination. After extracting, confirm you see `conf\` and `custom\` folders directly under `C:\hcpi`.

!!! tip "If you see a nested `opt\hcpi\` folder"
    Older exports (before the extraction guide was updated) preserved the full server path, so you'd end up with `C:\hcpi\opt\hcpi\conf\` instead. If that happened, move `conf\` and `custom\` up so they sit directly under `C:\hcpi`, then delete the empty `opt\` folder. Re-exports from the current guide won't have this problem.

Create the log directory:

**PowerShell:**
```powershell
Set-Location C:\hcpi
New-Item -ItemType Directory -Path log
```

Then clone Odoo 18. It's a separate command so you can re-run it on its own if the download fails partway (this can take a few minutes depending on your connection):

**PowerShell:**
```powershell
Set-Location C:\hcpi
git clone --depth 1 --branch 18.0 https://github.com/odoo/odoo.git
```

You should now have:

```
C:\hcpi\
├── conf\          # Configuration files (from hcpi-files.zip)
├── custom\        # Contains HCPI module (from hcpi-files.zip)
│   └── HCPI\      # Main HCPI module
├── log\           # Log files (created)
├── odoo\          # Odoo 18 codebase (cloned)
└── venv\          # Python virtual environment (will create next)
```

## Step 9: Set Up Python Virtual Environment

Open Terminal/Powershell in `C:\hcpi`:

```cmd
cd C:\hcpi
python -m venv venv
venv\Scripts\activate
```

Your prompt should now show `(venv)`.

Install dependencies:

```cmd
python -m pip install --upgrade pip
pip install wheel
pip install numpy
pip install -r odoo\requirements.txt
```

!!! tip "Installation Issues"
    If you encounter errors installing some packages, you may need to install Visual C++ Build Tools from Microsoft.

!!! info "NumPy Requirement"
    NumPy is required for HCPI but not included in Odoo's default requirements, so we install it separately.

## Step 10: Open the Project in an IDE

Editing `hcpi.conf` with Notepad works, but for the rest of your HCPI work — exploring modules, debugging, following the code — a proper IDE is much nicer. Set it up once now so the next step (and everything after) is easier. VS Code is the recommended choice; PyCharm is a fine alternative if you already know it.

### VS Code (recommended)

VS Code is fully free and works natively on Windows. All features are available without a paid tier.

1. **Install VS Code** if you don't have it: [code.visualstudio.com](https://code.visualstudio.com/). During installation, **tick "Add to PATH"** — this is what lets you launch VS Code with `code .` from any terminal. It's on by default but worth confirming.
2. **Open the project**. From PowerShell in `C:\hcpi`:

    ```powershell
    code .
    ```

    Or launch VS Code from the Start menu, then File → Open Folder → `C:\hcpi`.

3. **Install the Python extension.** Open the Extensions panel (Ctrl+Shift+X — circled in red below), search for "Python", and install the Microsoft one (highlighted in the rectangle):

    ![Install Python extension in VS Code](../img/install_python_vscode.png)

4. **Select the interpreter.** Press `Ctrl+Shift+P` and start typing `Python: Select Interpreter`:

    ![Select Python interpreter command](../img/select_interpreter1.png)

    In the list you'll typically see your venv shown as something like `.\venv\Scripts\python.exe` (a relative path) — pick that:

    ![Pick the venv from the interpreter list](../img/select_venv.png)

    If you don't see it, click "Enter interpreter path..." and paste the full path:

    ```
    C:\hcpi\venv\Scripts\python.exe
    ```

5. **Install the Odoo extension.** In Extensions, search for *Odoo* and install the one published by **Odoo S.A.** (the official one — free, despite the publisher name). It adds autocomplete and navigation for Odoo models, fields, XML views, and decorators — very useful once you start editing modules. You won't need it to just run HCPI, but you'll want it before making your first code change.

??? note "`code .` returns 'not recognized'"
    Means VS Code isn't on your PATH. Try these in order:

    1. **Close and reopen PowerShell.** PATH changes only take effect in new shells.
    2. **Reinstall VS Code** and make sure "Add to PATH" is ticked in the installer. (The installer also offers to do this without a full reinstall — re-run the installer and pick *Modify*.)
    3. **Add it manually**: open System Properties → Environment Variables → edit `Path` → add `C:\Users\<you>\AppData\Local\Programs\Microsoft VS Code\bin`. Then open a fresh PowerShell window.
    4. **As a fallback**, skip `code .` entirely — open VS Code from the Start menu and use File → Open Folder → `C:\hcpi`.

??? note "PyCharm alternative"
    PyCharm Community is free, but several features useful for Odoo work are **Professional-only** (~$99/year):

    - Database tools
    - HTTP client
    - Remote development

    Community edition is fine for editing code and managing the interpreter; Professional adds nicer DB/HTTP integration.

    1. **Install PyCharm** (Community or Professional): [jetbrains.com/pycharm](https://www.jetbrains.com/pycharm/).
    2. **Open the project**: File → Open → `C:\hcpi`.
    3. **Add the interpreter**: File → Settings → Project: hcpi → Python Interpreter → gear icon → Add → *System Interpreter* → browse to `C:\hcpi\venv\Scripts\python.exe`.

## Step 11: Configure Odoo

The configuration file is already provided in `C:\hcpi\conf\hcpi.conf`. Review and update the following settings:

Edit with a text editor (Notepad, VS Code, etc.) and adjust these key settings:

**Key settings to update:**

```ini
[options]
; Windows requires explicit host
db_host = localhost
db_port = 5432

; Set this to match the PostgreSQL password you set for the hcpi user in Step 6
db_password = your_secure_password

; UPDATE THESE PATHS if you installed to a different location - use backslashes (\)
addons_path = C:\hcpi\odoo\addons,C:\hcpi\custom\HCPI
logfile = C:\hcpi\log\hcpi.log

; Change this if port 9201 is already in use
http_port = 9201
```

!!! warning "Important Windows-Specific Settings"
    - **db_password**: Must match the PostgreSQL password you set for the `hcpi` user in Step 6
    - **db_host**: Must be `localhost` on Windows (unlike Linux where it can be `False`)
    - **Paths**: Must use backslashes (`\`), not forward slashes (`/`)
    - **Paths location**: Update `addons_path` and `logfile` if you chose a different installation location
    - **http_port**: Change if port 9201 is already in use

!!! info "About `admin_passwd`"
    The config from your export already has an `admin_passwd` set — that's the master password for database management operations (backup, restore, drop via Odoo's database manager UI). Leave it as-is to keep using the source instance's value, or change it here if you want a different one. It's not related to user logins, just DB-level admin actions.

## Step 12: Set Up Your Data

Pick one of the two options below. Both are equally valid — your choice depends on whether you have an existing database to clone from, or want to start fresh.

### Option A: Restore from an existing instance (recommended if you have the files)

A full restore has **two parts** that must both be done: the database, then the filestore. Missing the filestore will leave broken attachments and images in the restored instance.

#### A1: Restore the database

Use the `hcpi.dump` file produced by the [extraction guide](../extraction/linux-export.md) (PostgreSQL custom format). In Command Prompt or PowerShell:

```cmd
pg_restore -U hcpi -d hcpi --no-owner --no-privileges -j 4 C:\hcpi-export\hcpi.dump
```

Adjust the path if you saved the export somewhere else. Enter the `hcpi` user password when prompted. `pg_restore` handles cross-platform differences more gracefully than `psql`, so you should see few or no errors.

??? note "Re-running the restore / getting 'already exists' errors"
    `pg_restore` expects an empty database. If you're running it a second time — because the first run failed partway, or you want to start over — you'll see errors like `relation "..." already exists` or `constraint "..." already exists`, because the tables from the first attempt are still there.

    The cleanest fix is to drop the database and recreate it empty, then re-run the restore:

    **Step 1.** Make sure Odoo isn't running (it holds a DB connection that blocks the drop). If Odoo is running in another terminal, stop it with `Ctrl+C`.

    **Step 2.** Drop the old database (enter the postgres user password when prompted):

    ```cmd
    dropdb -U postgres hcpi
    ```

    **Step 3.** Recreate it empty, owned by the hcpi user:

    ```cmd
    createdb -U postgres -O hcpi hcpi
    ```

    **Step 4.** Re-run the restore:

    ```cmd
    pg_restore -U hcpi -d hcpi --no-owner --no-privileges -j 4 C:\hcpi-export\hcpi.dump
    ```

    If `dropdb` fails with "database is being accessed by other users", some process still has a connection. Close any running `psql`/pgAdmin sessions, stop Odoo, and try again. As a last resort, force-close other connections from pgAdmin 4 → right-click the database → Disconnect Database, or run:

    ```cmd
    psql -U postgres -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='hcpi' AND pid <> pg_backend_pid();"
    dropdb -U postgres hcpi
    ```

    After the fresh restore, also **wipe the filestore** if you're re-importing from scratch — otherwise you'll have orphan files from the previous attempt mixed in with the new ones. In File Explorer, delete `%LOCALAPPDATA%\Odoo\filestore\hcpi`, then re-do the A2 filestore step below.

??? note "If you have a plain `hcpi.sql` file instead (legacy)"
    Older exports sometimes ship as a plain SQL file inside `hcpi-db.zip`. The current extraction flow does **not** produce this — if you have one, it's from an older process.

    ```cmd
    psql -U hcpi -d hcpi -v ON_ERROR_STOP=0 -f path\to\hcpi.sql 2> restore_errors.log
    ```

    The `-v ON_ERROR_STOP=0` flag prevents the restore from stopping on minor errors (common when restoring Linux dumps on Windows). Errors are logged to `restore_errors.log` for review. Most errors related to permissions or Linux-specific features can be safely ignored.

#### A2: Restore the filestore

On native Windows, Odoo stores the filestore under your user profile at `%LOCALAPPDATA%\Odoo\filestore\<db_name>`. For most users that expands to `C:\Users\<YourName>\AppData\Local\Odoo\filestore\hcpi`.

The folder doesn't exist yet — Odoo creates it on first use. Since you're restoring before that first use, **you need to create it manually**. The easiest way is PowerShell, which creates any missing parent folders for you.

### Option 1: PowerShell (recommended — one command does everything)

Open PowerShell and run:

```powershell
$dest = "$env:LOCALAPPDATA\Odoo\filestore"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Expand-Archive -Path "C:\hcpi-export\hcpi-filestore.zip" -DestinationPath $dest
Get-ChildItem $dest
```

This creates the folder, extracts the zip, and lists the result. You should see one line: a folder named `hcpi`.

Adjust the `-Path` if your zip is somewhere other than `C:\hcpi-export\`.

### Option 2: File Explorer (step by step)

**Step 1 — Open your Local AppData folder.** Paste this into File Explorer's address bar and press Enter:

```
%LOCALAPPDATA%
```

This folder always exists — it's `C:\Users\<YourName>\AppData\Local`.

**Step 2 — Create the `Odoo` folder.** Inside `AppData\Local`, right-click in empty space → **New** → **Folder** → name it `Odoo`. Open it.

**Step 3 — Create the `filestore` folder.** Inside `Odoo`, right-click → **New** → **Folder** → name it `filestore`. Open it.

**Step 4 — Extract `hcpi-filestore.zip` into `filestore`.** In File Explorer, right-click `hcpi-filestore.zip` (in `C:\hcpi-export\`) → **Extract All...** → for the destination, paste `%LOCALAPPDATA%\Odoo\filestore\` (Windows will expand it). Click Extract.

### Confirm the result

Inside `%LOCALAPPDATA%\Odoo\filestore\` you should now see a folder named `hcpi` (or whatever your `db_name` is). Open it — you should see many two-character folders (`01`, `02`, `03`, ...) and a `checklist` file. That's the correct structure.

!!! warning "Path must match db_name exactly"
    The folder inside `filestore\` must be named exactly the same as `db_name` in `hcpi.conf`. If the extracted folder is `hcpi` but you named your database `ug_hcpi`, rename the folder or Odoo won't find the attachments. (Older exports may extract with wrapper folders like `home\hcpi\.local\share\Odoo\filestore\hcpi\` — if you see that, move the inner `hcpi` folder directly under `%LOCALAPPDATA%\Odoo\filestore\`.)

### Option B: Start with Empty Instance

Skip this step entirely — no database restore, no filestore. Odoo will initialize a fresh empty database when you first run it with the `-i HCPI` flag (see Step 13).

## Step 13: Start HCPI

Open PowerShell in `C:\hcpi` and activate the venv:

```powershell
cd C:\hcpi
venv\Scripts\activate
```

### Start HCPI

Whether you restored from a dump (Step 12 Option A) or will start fresh in a moment, the command to run HCPI is the same:

```powershell
python odoo\odoo-bin -c conf\hcpi.conf
```

Every time you want to run HCPI going forward, use this command.

??? note "Used empty database (Option B)? Do this once first"
    If you skipped the restore step and are starting with an empty database, you need a one-time initialization command to install the HCPI module into the fresh DB:

    ```powershell
    python odoo\odoo-bin -c conf\hcpi.conf -i HCPI --stop-after-init
    ```

    `-i HCPI` tells Odoo to *install* the HCPI module (and its dependencies) into the empty database. `--stop-after-init` runs the install and exits cleanly. Run this once, then start HCPI normally with the command above.

!!! info "First-time startup can take 1–3 minutes"
    On the very first run, Odoo builds asset bundles (JS/CSS) and populates base data. Subsequent starts are much faster. Watch the terminal for the line `HTTP service (werkzeug) running on ... port 9201` — that's your cue it's ready.

??? note "`odoo-bin` fails to start — common causes"
    Read the last lines of the traceback carefully; the error type below usually tells you what's wrong.

    **`psycopg2.OperationalError: FATAL: password authentication failed for user "hcpi"`**
    `db_password` in `hcpi.conf` doesn't match what PostgreSQL has. Reset it — in `psql -U postgres`:

    ```sql
    ALTER USER hcpi WITH PASSWORD 'your_secure_password';
    ```

    Then make sure the exact same string is in `conf\hcpi.conf`.

    **`could not connect to server: ... localhost:5432`**
    PostgreSQL isn't running. Open Services (Win+R → `services.msc`), find `postgresql-x64-15` (version may vary), and start it.

    **`psycopg2.OperationalError: ... Peer authentication failed`**
    You somehow have `db_host = False` in the config. On Windows this won't work — set `db_host = localhost` and `db_port = 5432`.

    **`ImportError: No module named '...'` / `ModuleNotFoundError`**
    Your venv isn't active, or a dependency didn't install. Activate the venv (`venv\Scripts\activate` — your prompt should show `(venv)`) and re-run `pip install -r odoo\requirements.txt`.

    **`FileNotFoundError: ... HCPI`** or the module isn't found
    The `addons_path` in `hcpi.conf` is wrong. Confirm it uses backslashes and points to the right folders: `addons_path = C:\hcpi\odoo\addons,C:\hcpi\custom\HCPI`.

    **`queue_job` not found**
    The exported config has `server_wide_modules = base,web,queue_job`. Confirm `C:\hcpi\custom\HCPI\queue_job\` exists. If missing, the export is incomplete — re-run the extraction guide. As a workaround, you can temporarily remove `queue_job` from `server_wide_modules` in `hcpi.conf`.

    **Port 9201 already in use**
    Another instance is running, or something else is on that port. Find it: `Get-NetTCPConnection -LocalPort 9201` in PowerShell. Kill the owning process or change `http_port` in `hcpi.conf`.

## Step 14: Access HCPI

Open your web browser and navigate to:

```
http://localhost:9201
```

Or if you changed the port in the configuration, use that port instead.

!!! tip "Checking the Port"
    If you're unsure which port HCPI is running on, check the configuration file or look for a line in the logs that says "HTTP service (werkzeug) running on..."

!!! warning "First Load May Be Slow"
    The first time you access HCPI, the page may take 30-60 seconds to load as it initializes the interface. After this initial load, performance should be normal.

### Sign in

**If you restored from an existing database** (Option A in Step 12): sign in with the same credentials you use on the source instance's web version — the users from the source DB are in your restored copy.

**If you started empty** (Option B in Step 12): Odoo created a default admin user. Log in with:

- **Email**: `admin`
- **Password**: `admin`

Then immediately change it: top-right user menu → **My Profile** → **Preferences** → change the password. For a real deployment, also create a second admin-level user (Settings → Users & Companies → Users) so you don't get locked out if you lose the admin account.

### Fix Missing Icons

If you notice missing icons in the interface after installation:

**Before (missing icons):**

![Missing Icons](../img/no-icons.png)

**To fix:**

1. Click on your username in the top right
2. Go to **Settings** → **Activate the developer mode**
3. Then go to **Settings** → **Technical** → **User Interface** → **Regenerate Assets Bundles**
4. Refresh the page

**After (icons restored):**

![Icons Restored](../img/after-regeneration.png)

Alternatively, you can restart HCPI with the asset regeneration flag:

```cmd
python odoo\odoo-bin -c conf\hcpi.conf -u all --dev=all
```

## Creating a Startup Batch File

Create `C:\hcpi\start-hcpi.bat`:

```batch
@echo off
cd C:\hcpi
call venv\Scripts\activate
python odoo\odoo-bin -c conf\hcpi.conf
pause
```

Now you can double-click this file to start HCPI.

## Running as a Windows Service (Advanced)

For production-like setup, you can run HCPI as a Windows service using NSSM (Non-Sucking Service Manager):

1. Download NSSM from [nssm.cc](https://nssm.cc/download)
2. Extract and open Command Prompt as Administrator
3. Navigate to NSSM directory
4. Run:

```cmd
nssm install HCPI
```

Configure:
- Path: `C:\hcpi\venv\Scripts\python.exe`
- Startup directory: `C:\hcpi`
- Arguments: `odoo\odoo-bin -c conf\hcpi.conf`

## Troubleshooting

### Python Not Found

Ensure Python is in your PATH:
```cmd
where python
```

### PostgreSQL Connection Errors

Check if PostgreSQL service is running:
- Open Services (Win+R → `services.msc`)
- Find "postgresql-x64-15" (version may vary)
- Ensure it's running

### Database Connection Failed

Make sure `db_host` is set to `localhost` in the config file (not `False` like in Linux).

### Module Import Errors

Ensure virtual environment is activated:
```cmd
venv\Scripts\activate
```

### Port 9201 Already in Use

Change the port in `conf\hcpi.conf`:
```ini
http_port = 9202
```

### Check Logs

Open `C:\hcpi\log\hcpi.log` in a text editor to see error messages.

### Module Not Found

Verify the addons_path in the config file uses Windows paths:

```ini
addons_path = C:\hcpi\odoo\addons,C:\hcpi\custom\HCPI
```

### Antivirus Interference

Some antivirus software may block Odoo. Add `C:\hcpi` to your antivirus exclusions if you experience issues.

### Path Issues

Make sure all paths in `hcpi.conf` use backslashes (`\`), not forward slashes (`/`).

## Known Limitations on Windows

- Some Odoo modules may have compatibility issues with Windows
- Performance may be lower compared to Linux
- File path handling differences may cause minor issues
- Some system-level features may not work identically to Linux

## Next Steps

HCPI is now running on your machine. To start working with the code:

➡️ **[Understanding the Codebase](../understanding-the-codebase/index.md)** — a map of what's where and how to find things.

Then:

➡️ **[Making Your First Edits](../first-edits/index.md)** — small, safe changes to build confidence.

For further reading and setup:

- Review the [Odoo 18 documentation](https://www.odoo.com/documentation/18.0/)
- For a more production-like environment, consider using [Windows WSL](windows-wsl.md)
- Configure HCPI modules for your needs
- Set up regular database backups using pgAdmin or command-line tools
- Configure user accounts and permissions in HCPI
