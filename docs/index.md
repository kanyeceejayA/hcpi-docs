# HCPI Documentation

Welcome to the HCPI (Harmonized Consumer Price Index) documentation. HCPI is a comprehensive system for collecting and analyzing price data, used by EAC Statistics bodies.

## What is HCPI?

HCPI is a Cost Price Index system built on [Odoo 18](https://www.odoo.com/documentation/18.0/) that enables statistical organizations to:

- Collect price data efficiently
- Analyze pricing trends
- Generate statistical reports
- Manage field data collection operations

## Live Demo

You can explore a live instance of the Ugandan version at [https://uboscpi.ubos.org/odoo](https://uboscpi.ubos.org/odoo)

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

## Download Files

Installation files are available at:
[http://https://statistics.ubos.org/shares/d/z_M6k4Jya_lxN6lWX5Wz_w](http://https://statistics.ubos.org/shares/d/z_M6k4Jya_lxN6lWX5Wz_w)

- **hcpi-files.zip**: Contains the `conf` and `custom` folders with HCPI modules
- **hcpi.dump**: PostgreSQL database dump in custom format (optional — only needed if you want sample data). Restored with `pg_restore`.

!!! info "Empty Instance"
    You don't need to restore the database if you want to start with an empty HCPI instance. Odoo will initialize a fresh database for you.

!!! warning "Files at this link are Uganda's test data"
    To set HCPI up for another country, produce your own export from that country's server first — see [Exporting HCPI from a Linux Server](extraction/linux-export.md).

## Need Help?

If you encounter issues during installation or usage, please refer to the specific installation guide for your platform or consult the [Odoo 18 documentation](https://www.odoo.com/documentation/18.0/).
