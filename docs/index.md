# HCPI Documentation

Welcome to the HCPI (Harmonized Consumer Price Index) documentation. HCPI is a comprehensive system for collecting and analyzing price data, used by EAC Statistics bodies.

## What is HCPI?

HCPI is built on [Odoo 18](https://www.odoo.com/documentation/18.0/) and enables statistical organizations to:

- Collect price data efficiently
- Analyze pricing trends
- Generate statistical reports
- Manage field data collection operations

## Live Version

You can explore a live instance of the Ugandan version at [https://uboscpi.ubos.org/odoo](https://uboscpi.ubos.org/odoo). Other versions for other countries are hosted on their specific links

## Getting Started

To get started with HCPI, follow these steps:

1. Review the [Prerequisites](getting-started/prerequisites.md) to ensure your system meets the requirements
2. Choose your installation method:
   - [Linux Server Installation](installation/linux.md) - Recommended for production
   - [Windows WSL Installation](installation/windows-wsl.md) - Good for development
   - [Windows Native Installation](installation/windows-native.md) - For testing (may have minor differences from production)

## System Overview

HCPI is built on Odoo 18 and consists of:

- **Custom HCPI Module**: Contains the price index-specific functionality
- **Odoo Core**: The underlying framework (Odoo 18)
- **PostgreSQL Database**: For data storage
- **Python Virtual Environment**: For dependency management

## Getting the Installation Files

For a real deployment, you produce the installation files from your own country's HCPI server. The [Exporting HCPI from a Linux Server](extraction/linux-export.md) guide walks you through it, and you'll end up with:

- **`hcpi-files.zip`** — the `conf` and `custom` folders (application code and configuration)
- **`hcpi.dump`** — PostgreSQL database dump in custom format (optional — skip for an empty instance)
- **`hcpi-filestore.zip`** — uploaded attachments and images (optional — skip for an empty instance)

!!! info "Empty Instance"
    You don't need the database dump or filestore if you want to start with an empty HCPI instance. Odoo will initialize a fresh database for you on first run.

??? note "No access to a source server? Use Uganda's test files as a fallback"
    [https://statistics.ubos.org/shares/d/z_M6k4Jya_lxN6lWX5Wz_w](https://statistics.ubos.org/shares/d/z_M6k4Jya_lxN6lWX5Wz_w) hosts a zipped set from Uganda's instance. Intended for evaluation and reference only — use them if you want to try HCPI before setting it up for your own country. For a real deployment, prefer files exported from your country's own server.

## Need Help?

If you encounter issues during installation or usage, please refer to the specific installation guide for your platform or consult the [Odoo 18 documentation](https://www.odoo.com/documentation/18.0/).
