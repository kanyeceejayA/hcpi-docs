# Server Setup

Before we touch any HCPI code, every trainee needs a base operating environment. This page covers the *choice* of host, the directory layout you're aiming for, and the user account assumptions everything that follows will make.

## Pick a host

HCPI runs on Linux. For training, you have three options:

| Environment | When to use it | Trade-offs |
|---|---|---|
| **Ubuntu Linux server** | Production, real deployments | Production-grade, no surprises |
| **Windows + WSL2 (Ubuntu)** | Development on a Windows laptop | Closest to Linux on Windows; recommended for training |
| **Windows native** | Testing, demos | Works, but some Odoo features behave slightly differently — not for production |

For the training, **Windows + WSL2** is what most trainees will use. If you're following along on a real Linux server, the steps are nearly identical — paths and `apt` commands are the same.

!!! tip "Recommended for training"
    Windows laptop → install WSL2 → install Ubuntu. You get a real Linux shell on your laptop without dual-booting. The [Windows WSL Installation](../../installation/windows-wsl.md) guide is what you'll follow.

System requirements are documented in [Prerequisites](../../getting-started/prerequisites.md) — minimum 4 GB RAM, 10 GB free disk.

## The directory layout you're aiming for

By the end of Day 1, your filesystem (inside Linux/WSL) should look like this:

```
/opt/hcpi/
├── conf/          ← hcpi.conf and friends
├── custom/        ← Your country's HCPI modules
│   └── HCPI/
├── log/           ← Runtime logs
├── odoo/          ← Odoo 18 framework (cloned from GitHub)
└── venv/          ← Python virtual environment
```

We use `/opt/hcpi` by convention. Nothing technically prevents you from putting it elsewhere — `/home/<you>/hcpi`, or any other directory you own — but every command in the docs assumes `/opt/hcpi`, and the configuration file (`addons_path`, `logfile`) will have to be updated if you deviate.

!!! info "Why /opt?"
    On Linux, `/opt` is the conventional location for "optional add-on application software". It's a clean choice for apps that aren't managed by the system package manager. Putting HCPI here keeps it separate from your personal files (`/home`) and from system files (`/usr`).

## User account assumptions

You don't run HCPI as `root`. Throughout the docs:

- **Your Linux user** (whatever you called yourself when you set up WSL/Ubuntu) is the user that owns `/opt/hcpi` and runs the `odoo-bin` process. After creating the directory, we chown it to you:

    ```bash
    sudo chown $USER:$USER /opt/hcpi
    ```

- **`hcpi` is a PostgreSQL user** — entirely separate from your Linux user. It's the role used to connect to the database. Don't confuse the two.
- **`postgres` is also a PostgreSQL system user**, created when you install PostgreSQL. You'll use it to bootstrap the `hcpi` role (via `sudo -u postgres ...`), then you won't touch it again.

This three-user separation matters for one specific reason that bites people on first run: PostgreSQL's default authentication mode (peer auth) tries to match your Linux username to a PostgreSQL role of the same name. Your Linux username is *not* `hcpi`, so peer auth fails and the connection is refused. The fix — forcing TCP+password auth in `hcpi.conf` — is covered on the [PostgreSQL Setup](postgresql.md) and [Configuration File](configuration.md) pages.

## Networking and ports

HCPI listens on **port 9201** by default (not Odoo's stock 8069 — that's deliberate, to avoid collisions if you run multiple Odoo instances). You'll reach it at:

```
http://localhost:9201
```

For training nothing else is involved — no firewall rules, no reverse proxy, no TLS. Production deployments add HTTPS via Apache or Nginx; see [HTTPS with Apache](../../next-steps/https-apache.md) or [HTTPS with Nginx](../../next-steps/https-nginx.md) when you're ready for that.

If 9201 is already in use on your machine — common if you've run Odoo before — pick another port in `hcpi.conf`:

```ini
http_port = 9202
```

## Practical steps

Now actually do the setup. Follow the install guide that matches your machine:

- **[Windows WSL Installation](../../installation/windows-wsl.md)** — Steps 1–6 cover everything on this page (WSL, Ubuntu, package updates, directory creation)
- **[Linux Server Installation](../../installation/linux.md)** — Steps 1, 4
- **[Windows Native Installation](../../installation/windows-native.md)** — if you're going that route

Come back here when `/opt/hcpi` exists and you own it.

➡️ Next: [Installing Dependencies](dependencies.md)
