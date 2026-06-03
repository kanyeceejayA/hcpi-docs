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

## Getting the Code

Three sources to choose from depending on your situation:

1. **Clone the EAC upstream repo** — canonical source code, empty instance. Right starting point for new deployments. Repo: <https://github.com/East-African-Community-HCPI/HCPI> — request access from [mkakinyi@eachq.org](mailto:mkakinyi@eachq.org).
2. **Export from a running country server** — code plus database dump plus filestore. For migrating an existing instance to a new machine.
3. **Uganda's published test files** — sample code + data for evaluation and training only.

See **[Getting the Code](getting-started/getting-the-code.md)** for the full walkthrough of all three options, including the minimal `hcpi.conf` template you'll need if you're cloning the upstream repo.

## Need Help?

If you encounter issues during installation or usage, please refer to the specific installation guide for your platform or consult the [Odoo 18 documentation](https://www.odoo.com/documentation/18.0/).

## User Manual

Download the user manual here: [User Manual](User_Manual_EAC_CPI_Software_PS_March_2023.pdf)