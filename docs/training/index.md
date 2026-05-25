# Back-end Training Programme

This section follows the official **HCPI Back-end Training** programme. It exists alongside the installation guides — those tell you *what to type*; the training pages tell you *what's happening and why*.

## How to use this section

- **If you're following a trainer**, read each day's pages in order. Each topic has a "Practical steps" block at the bottom that links out to the installation guide for the exact commands.
- **If you're self-studying**, the same pages still work — but plan to spend more time on the conceptual sections and come back to the practical steps when you sit down at a machine.

## Programme at a glance

| Day | Theme |
|---|---|
| [Day 1](day1/introduction.md) | Setup & Foundations — system overview, server, dependencies, PostgreSQL, virtualenv, IDE, config, user admin |
| Day 2 | Python for HCPI & building a custom module (Models, Views, Controllers, Access Rights) — *coming soon* |
| Day 3 | Models, fields, relationships, ORM CRUD — *coming soon* |
| Day 4 | XML views (Form, Tree, Kanban, Graph) — *coming soon* |
| Day 5 | Advanced views, QWeb, OWL — *coming soon* |
| Day 6 | Search views, one2many rendering, tree actions — *coming soon* |
| Day 7 | Security & access control — *coming soon* |
| Day 8 | HCPI project structure: Coicop & Location modules — *coming soon* |
| Day 9 | Outlets, Items, UoMs, Price Relatives — *coming soon* |
| Day 10 | Data collection, validation (Tukey), processing — *coming soon* |
| Day 11 | Data importation, cleaning, templates — *coming soon* |
| Day 12 | Mobile app source, HCPI–Flutter API, custom APIs, Q&A — *coming soon* |

## Day 1 contents

The day starts with concepts and ends with a running HCPI on every trainee's machine. Topics:

1. [Introduction to HCPI](day1/introduction.md) — system overview, architecture, what's on disk
2. [Server Setup](day1/server-setup.md) — picking a host OS, directory layout, user accounts
3. [Installing Dependencies](day1/dependencies.md) — the system packages HCPI needs and what each one is for
4. [PostgreSQL Setup](day1/postgresql.md) — installing, creating the HCPI user and database, authentication modes
5. [Virtual Environment & HCPI Install](day1/virtualenv-install.md) — why a venv, getting the code in place, Python deps
6. [Development Environment](day1/dev-environment.md) — VS Code with WSL, the Odoo extension, PyCharm alternative
7. [Configuration File](day1/configuration.md) — `hcpi.conf` walkthrough, `addons_path`, ports, DB connection
8. [User Administration](day1/user-administration.md) — adding users, resetting passwords, the master password, developer mode
