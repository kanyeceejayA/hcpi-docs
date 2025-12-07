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

## Download Required Files

Before starting installation, download the required files from:
[http://statistics.ubos.org/hcpishare](http://statistics.ubos.org/hcpishare)

1. **hcpi-code.zip** - Contains the HCPI application code
2. **hcpi-sql.zip** - Contains the PostgreSQL database dump (optional)

!!! tip "Database Restore"
    Only download and restore hcpi-sql.zip if you want to start with sample data. You can skip this for a fresh, empty instance.

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
