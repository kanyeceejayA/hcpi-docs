# About HCPI

## Overview

HCPI (Harmonized Consumer Price Index) is a comprehensive system designed for statistical organizations to collect, manage, and analyze price data for consumer price index calculations.

## Built on Odoo 18

HCPI is built on top of [Odoo 18](https://www.odoo.com/documentation/18.0/), a powerful open-source business application platform. This provides HCPI with:

- A robust framework for data management
- User authentication and access control
- Flexible reporting capabilities
- Mobile-friendly interface
- Extensive customization options

## Key Features

### Data Collection
- Streamlined price data entry
- Support for field data collection operations
- Mobile-responsive interface for on-the-go data entry

### Analysis & Reporting
- Price trend analysis
- Statistical calculations
- Customizable reports
- Data visualization tools

### Multi-Organization Support
- Used by EAC Statistics bodies
- Configurable for different organizational needs
- Multi-user access with role-based permissions

## Directory Structure

The HCPI installation consists of the following components:

```
/opt/hcpi/
├── conf/          # Configuration files
├── custom/        # Custom modules including HCPI
│   └── HCPI/      # Main HCPI module
├── log/           # Application logs
├── odoo/          # Odoo 18 core codebase
└── venv/          # Python virtual environment
```

## Technical Stack

- **Backend**: Python 3.10+
- **Framework**: Odoo 18
- **Database**: PostgreSQL 12+
- **Frontend**: Web-based (HTML, JavaScript, CSS)
- **Reporting**: wkhtmltopdf for PDF generation

## Live Example

A live deployment of HCPI can be accessed at:
[https://uboscpi.ubos.org/odoo](https://uboscpi.ubos.org/odoo)

This is the Ugandan version managed by UBOS (Uganda Bureau of Statistics).

## Download Files

Installation files are available at:
[http://statistics.ubos.org/hcpishare](http://statistics.ubos.org/hcpishare)

- **hcpi-code.zip**: Complete HCPI application code
- **hcpi-sql.zip**: Sample PostgreSQL database

## Use Cases

HCPI is designed for:

- National statistics offices
- Regional statistical organizations
- Research institutions studying price trends
- Economic analysis organizations

## Support & Resources

- **Odoo Documentation**: [https://www.odoo.com/documentation/18.0/](https://www.odoo.com/documentation/18.0/)
- **Installation Guides**: See the Installation section in this documentation
- **Getting Started**: Review the [Prerequisites](getting-started/prerequisites.md) page

## License

HCPI is built on Odoo, which is available under the LGPL v3 license. Please refer to the Odoo documentation for specific licensing details.
