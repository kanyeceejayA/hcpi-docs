# Back-end Training Programme

This section follows the official **HCPI Back-end Training** programme. It exists alongside the installation guides — those tell you *what to type*; the training pages tell you *what's happening and why*.

## How to use this section

- **If you're following a trainer**, read each day's pages in order. Each topic has a "Practical steps" block at the bottom that links out to the installation guide for the exact commands.
- **If you're self-studying**, the same pages still work — but plan to spend more time on the conceptual sections and come back to the practical steps when you sit down at a machine.

## Programme at a glance

| Day | Theme |
|---|---|
| [Day 1](day1/introduction.md) | Setup & Foundations — system overview, server, dependencies, PostgreSQL, virtualenv, IDE, config, user admin |
| Day 2 | [Python for HCPI](day2/python-intro.md) — language essentials for devs from other languages (~1h) + the warm-up [Your First Module](../first-edits/your-first-module.md) hand-roll |
| Day 3 | [Field Reports — Part 1](module/part1-models.md): models, field types, relationships (M2O, O2M, M2M, hierarchical), inheritance, computed fields, ORM CRUD |
| Day 4 | [Field Reports — Part 2 (morning)](module/part2-views.md): all view types (List, Form, Kanban, Graph, Pivot, Calendar), search filters & group bys |
| Day 4½ | Part 2 (afternoon): inline One2many, list decorations, server actions, QWeb PDF reports, brief OWL look |
| Day 5 | [Field Reports — Part 3](module/part3-security.md): groups, ACLs, record rules, sequences, `mail.thread` chatter, workflow validation, view inheritance |
| Day 6 | HCPI project structure: Coicop & Location modules — *coming soon* |
| Day 7 | Outlets, Items, UoMs, Price Relatives — *coming soon* |
| Day 8 | Data collection, validation (Tukey), processing — *coming soon* |
| Day 9 | Data importation, cleaning, templates — *coming soon* |
| Day 10 | Mobile app source, HCPI–Flutter API, custom APIs, Q&A — *coming soon* |

!!! info "Compressed schedule"
    The original draft programme spread the language/module/views/security content across Days 2–7 (six days). This documentation tracks a **2½-day condensed version**: one hour of Python, then three guided parts of an iterating module that covers Days 3–7 of the original programme.

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
