# Windows WSL Installation

This guide shows you how to install HCPI on Windows using WSL (Windows Subsystem for Linux). This setup provides a Linux environment on Windows and is recommended for development work.

!!! info "What is WSL?"
    WSL allows you to run a Linux distribution alongside your Windows installation. This provides better compatibility with Odoo and HCPI compared to native Windows installation.

## Step 1: Install WSL

Open PowerShell as Administrator and run:

```powershell
wsl --install
```

This installs WSL with Ubuntu by default. Restart your computer when prompted.

!!! tip "Existing WSL Installation"
    If you already have WSL installed, you can install Ubuntu with:
    ```powershell
    wsl --install -d Ubuntu
    ```

## Step 2: Set Up Ubuntu

After restart, Ubuntu will open automatically. Create your Linux username and password when prompted.

!!! warning "Remember Your Password"
    You'll need this password for sudo commands. Make sure to remember it!

## Step 3: Update Ubuntu Packages

In your Ubuntu terminal:

```bash
sudo apt update
sudo apt upgrade -y
```

## Step 4: Install System Dependencies

Install required packages:

```bash
sudo apt install -y git python3 python3-pip python3-dev python3-venv \
    postgresql postgresql-contrib libpq-dev \
    build-essential libssl-dev libffi-dev libxml2-dev libxslt1-dev \
    zlib1g-dev libjpeg-dev libsasl2-dev libldap2-dev \
    node-less npm wkhtmltopdf unzip
```

## Step 5: Configure PostgreSQL

Start PostgreSQL service:

```bash
sudo service postgresql start
```

Create a PostgreSQL user and database:

```bash
sudo -u postgres createuser -s hcpi
sudo -u postgres psql -c "ALTER USER hcpi WITH PASSWORD 'your_secure_password';"
sudo -u postgres createdb -O hcpi hcpi
```

!!! tip "Database User"
    We're using `hcpi` as both the database name and username. You can choose different names, but update the configuration file accordingly.

!!! tip "Auto-start PostgreSQL"
    PostgreSQL doesn't auto-start in WSL. You'll need to run `sudo service postgresql start` each time you open WSL, or add it to your `.bashrc`:
    ```bash
    echo "sudo service postgresql start" >> ~/.bashrc
    ```

## Step 6: Create Directory Structure

Create the HCPI directory:

```bash
sudo mkdir -p /opt/hcpi
sudo chown $USER:$USER /opt/hcpi
cd /opt/hcpi
```

!!! warning "Custom Installation Path"
    If you choose a different path than `/opt/hcpi`, you'll need to update the paths in the configuration file (see Step 8).

## Step 7: Transfer and Set Up Files

### Option A: Access Windows Files from WSL

Your Windows drives are accessible at `/mnt/c/`, `/mnt/d/`, etc.

If you've downloaded the files to your Windows Downloads folder:

```bash
cd /opt/hcpi

# Copy and extract HCPI files (contains conf and custom folders)
cp /mnt/c/Users/YourWindowsUsername/Downloads/hcpi-files.zip .
unzip hcpi-files.zip

# Clone Odoo 18
git clone --depth 1 --branch 18.0 https://github.com/odoo/odoo.git

# Create log directory
mkdir -p log
```

### Option B: Download Directly in WSL

If you can access the files via URL:

```bash
cd /opt/hcpi

# Download and extract HCPI files
wget http://statistics.ubos.org/hcpishare/hcpi-files.zip
unzip hcpi-files.zip

# Clone Odoo 18
git clone --depth 1 --branch 18.0 https://github.com/odoo/odoo.git

# Create log directory
mkdir -p log
```

After this, you should have:

```
/opt/hcpi/
├── conf/          # Configuration files (from hcpi-files.zip)
├── custom/        # Contains HCPI module (from hcpi-files.zip)
│   └── HCPI/      # Main HCPI module
├── log/           # Log files (created)
├── odoo/          # Odoo 18 codebase (cloned)
└── venv/          # Python virtual environment (will create next)
```

## Step 8: Set Up Python Virtual Environment

Create and activate the virtual environment:

```bash
cd /opt/hcpi
python3 -m venv venv
source venv/bin/activate
```

Install Python dependencies:

```bash
pip install --upgrade pip
pip install wheel
pip install numpy
pip install -r odoo/requirements.txt
```

!!! info "NumPy Requirement"
    NumPy is required for HCPI but not included in Odoo's default requirements, so we install it separately.

## Step 9: Configure Odoo

The configuration file is already provided in `/opt/hcpi/conf/hcpi.conf`. Review and update the following settings:

```bash
nano /opt/hcpi/conf/hcpi.conf
```

**Key settings to update:**

```ini
[options]
; Change this to a strong password for database management operations
admin_passwd = your_strong_admin_password

; Set this to match the PostgreSQL password from Step 5
db_password = your_secure_password

; UPDATE THESE PATHS if you installed to a different location than /opt/hcpi
addons_path = /opt/hcpi/odoo/addons,/opt/hcpi/custom/HCPI
logfile = /opt/hcpi/log/hcpi.log

; Change this if port 9201 is already in use
http_port = 9201
```

!!! warning "Important"
    - **admin_passwd**: Used for database management operations - choose a strong password
    - **db_password**: Must match the PostgreSQL password you created in Step 5
    - **Paths**: Update `addons_path` and `logfile` if you chose a different installation location
    - **http_port**: Change if port 9201 is already in use

!!! tip "Save in Nano"
    Press `Ctrl+X`, then `Y`, then `Enter` to save and exit nano.

## Step 10: Restore Database (Optional)

### Option A: Start with Sample Data

If you want to work with existing data:

```bash
cd /opt/hcpi
cp /mnt/c/Users/YourWindowsUsername/Downloads/hcpi-db.zip .
unzip hcpi-db.zip
psql -U hcpi -d hcpi -f hcpi.sql
```

Enter the hcpi user password when prompted.

### Option B: Start with Empty Instance

Simply skip this step. Odoo will initialize the database when you first run it with the `-i HCPI` flag (see Step 11).

## Step 11: Start HCPI

Make sure PostgreSQL is running:

```bash
sudo service postgresql start
```

Start HCPI:

```bash
cd /opt/hcpi
source venv/bin/activate
python odoo/odoo-bin -c conf/hcpi.conf
```

For first-time setup with empty database:

```bash
python odoo/odoo-bin -c conf/hcpi.conf -u all --dev=all
```

This ensures icons and assets are properly generated on the first run.

!!! info "First-Time Startup"
    The first time you start HCPI, it may take a few minutes to initialize. Be patient during the initial load. Subsequent starts will be much faster.

## Step 12: Access HCPI

Open your Windows web browser and navigate to:

```
http://localhost:9201
```

!!! warning "First Load May Be Slow"
    The first time you access HCPI, the page may take 30-60 seconds to load as it initializes the interface. After this initial load, performance should be normal.

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

```bash
python odoo/odoo-bin -c conf/hcpi.conf -u all --dev=all
```

## Creating a Startup Script (Optional)

Create a convenient startup script:

```bash
nano ~/start-hcpi.sh
```

Add:

```bash
#!/bin/bash
sudo service postgresql start
cd /opt/hcpi
source venv/bin/activate
python odoo/odoo-bin -c conf/hcpi.conf
```

Make it executable:

```bash
chmod +x ~/start-hcpi.sh
```

Now you can start HCPI with:

```bash
~/start-hcpi.sh
```

## Accessing Files Between Windows and WSL

### From WSL to Windows
Windows drives are mounted at `/mnt/`:
- C: drive → `/mnt/c/`
- D: drive → `/mnt/d/`

### From Windows to WSL
Open File Explorer and enter in the address bar:
```
\\wsl$\Ubuntu\opt\hcpi
```

## Troubleshooting

### PostgreSQL Won't Start

```bash
sudo service postgresql status
sudo service postgresql restart
```

### Database Connection Issues

Test the database connection:

```bash
psql -U hcpi -d hcpi -c "SELECT version();"
```

### Permission Issues

If you get permission errors:

```bash
sudo chown -R $USER:$USER /opt/hcpi
```

### Python Module Errors

Ensure you're in the virtual environment:

```bash
source /opt/hcpi/venv/bin/activate
```

### Port Already in Use

Change the port in `/opt/hcpi/conf/hcpi.conf`:

```ini
http_port = 9202
```

### Module Not Found

Verify the addons_path in the config file:

```ini
addons_path = /opt/hcpi/odoo/addons,/opt/hcpi/custom/HCPI
```

### Check Logs

```bash
tail -f /opt/hcpi/log/hcpi.log
```

## Performance Tips

- Place project files in the Linux filesystem (`/opt/hcpi`) rather than accessing Windows filesystem (`/mnt/c/`) for better performance
- Allocate more RAM to WSL if needed by creating `.wslconfig` in your Windows user directory

## Next Steps

- Review the [Odoo 18 documentation](https://www.odoo.com/documentation/18.0/) for configuration options
- Set up your development environment
- Configure HCPI modules for your organization's needs
- Configure user accounts and permissions in HCPI
