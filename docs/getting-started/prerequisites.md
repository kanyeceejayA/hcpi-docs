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

## Get the Code

Three sources, depending on your situation:

| Source | Best for | Has data? |
|---|---|---|
| **Clone the EAC upstream repo** (request access from [mkakinyi@eachq.org](mailto:mkakinyi@eachq.org)) | New deployments, new country adoption | ❌ Empty instance |
| **Export from a country's HCPI server** ([extraction guide](../extraction/linux-export.md)) | Migrating an existing instance | ✅ Full data |
| **Uganda's test files** ([statistics.ubos.org](https://statistics.ubos.org/shares/d/z_M6k4Jya_lxN6lWX5Wz_w)) | Evaluation, training, reference | ✅ Sample data |

See **[Getting the Code](getting-the-code.md)** for the full walkthrough of all three options, including the minimal `hcpi.conf` template if you're cloning fresh from the upstream repo.

!!! tip "Not sure which one?"
    For most new users — including any country setting up HCPI from scratch — **clone the EAC upstream repo**. It's the canonical source code and you start with a clean instance.

## Knowledge Prerequisites

This documentation assumes you have:

- Basic command-line familiarity
- Basic understanding of Python environments
- Basic Linux/Unix command knowledge (for Linux and WSL installations)

## Next Steps

Once you've confirmed your system meets these requirements and obtained the code, proceed to the installation guide for your platform:

- [Linux Server Installation](../installation/linux.md)
- [Windows WSL Installation](../installation/windows-wsl.md)
- [Windows Native Installation](../installation/windows-native.md)
