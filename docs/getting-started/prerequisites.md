# Prerequisites

Before installing HCPI, ensure your system meets the following requirements.

## System Requirements

### For Linux and Windows WSL

- **Operating System**: Ubuntu 20.04 LTS or later (or compatible Linux distribution)
- **RAM**: Minimum 4GB (8GB recommended for production)
- **Disk Space**: At least 10GB free space
- **User Privileges**: Sudo access for package installation

### For Windows Native

- **Operating System**: Windows 10 or Windows Server 2016 or later
- **RAM**: Minimum 4GB (8GB recommended)
- **Disk Space**: At least 10GB free space

!!! warning "Production Deployment"
    Windows native installation is suitable for testing and development. For production deployments, use Linux or Windows WSL for better compatibility and performance.

## Software Dependencies

### Python

- **Version**: Python 3.10 or later
- Used for running Odoo and HCPI modules

### PostgreSQL

- **Version**: PostgreSQL 12 or later
- The database backend for HCPI

### Additional Tools

The following will be installed during the setup process:

- Git (for version control)
- pip (Python package manager)
- virtualenv (for Python virtual environment management)
- wkhtmltopdf (for PDF report generation)

## Get the Required Files

You'll need these files, produced from your own country's HCPI server:

1. **`hcpi-files.zip`** — the HCPI application code and configuration
2. **`hcpi.dump`** — the PostgreSQL database dump (optional — skip if you want to start empty)
3. **`hcpi-filestore.zip`** — uploaded attachments and images (optional — skip if you're starting empty)

To produce them, follow the [Exporting HCPI from a Linux Server](../extraction/linux-export.md) guide. It walks you through generating all three files in one pass.

!!! tip "Starting with an empty instance?"
    You only need `hcpi-files.zip`. Skip the database dump and filestore — Odoo will initialize a blank database when it first runs. This is the simplest starting point if you don't have existing data to migrate.

??? note "No access to a source server?"
    [https://statistics.ubos.org/shares/d/z_M6k4Jya_lxN6lWX5Wz_w](https://statistics.ubos.org/shares/d/z_M6k4Jya_lxN6lWX5Wz_w) hosts a zipped set from Uganda's instance. These are intended for testing or reference only. They can let you understand the basics of the HCPI system, but there might be some variations between that system and what your country is running. For a real deployment, prefer files exported from your country's own server.

### Which combination do I need?

- **Full clone with data**: all three files. Use this to migrate a working instance to a new machine.
- **Empty instance**: just `hcpi-files.zip`. Odoo creates a fresh database on first run.
- **Code only from a source server**: just `hcpi-files.zip`, produced via the extraction guide. Use this when you want another country's custom modules but no data.

## Knowledge Prerequisites

This documentation assumes you have:

- Basic command-line familiarity
- Basic understanding of Python environments
- Basic Linux/Unix command knowledge (for Linux and WSL installations)

## Next Steps

Once you've confirmed your system meets these requirements and downloaded the necessary files, proceed to the installation guide for your platform:

- [Linux Server Installation](../installation/linux.md)
- [Windows WSL Installation](../installation/windows-wsl.md)
- [Windows Native Installation](../installation/windows-native.md)
