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

You need two things:

1. **`hcpi-files.zip`** — the HCPI application code and configuration
2. **`hcpi.dump`** — a PostgreSQL database dump (optional, only if you want to start with existing data)

Plus optionally:

3. **`hcpi-filestore.zip`** — uploaded attachments and images from the source instance

### Where to get them

!!! warning "The files at the public link are Uganda's test data"
    [http://statistics.ubos.org/hcpishare](http://statistics.ubos.org/hcpishare) hosts an older zipped set from Uganda's instance, intended for testing or reference only.

    If you are setting HCPI up for another country, **do not use those files**. Produce your own export from your country's server first — see [Exporting HCPI from a Linux Server](../extraction/linux-export.md). That guide walks you through generating all three files above.

!!! tip "Starting empty"
    You can skip the database dump **and** the filestore entirely if you want a fresh, empty HCPI instance. You only need `hcpi-files.zip` (the code and configuration). Odoo will initialize a blank database when it first runs with the `-i HCPI --dev=all` flags — covered in the installation guides.

!!! info "Three useful combinations"
    - **Empty instance, any country**: just `hcpi-files.zip`. No source server needed — if you're setting up HCPI for the first time in a new country, this is the simplest starting point.
    - **Full clone with data**: `hcpi-files.zip` + `hcpi.dump` + `hcpi-filestore.zip`. Use this to migrate a working instance to a new machine.
    - **Code only from a source server**: `hcpi-files.zip` alone, produced via the extraction guide. Use this when you want another country's custom modules but no data.

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
