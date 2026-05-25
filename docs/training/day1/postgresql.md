# PostgreSQL Setup

PostgreSQL is HCPI's database. This page covers the bits trainees most often get wrong on first install: the difference between PostgreSQL users and Linux users, the two authentication modes, and how to verify your setup before moving on.

## Why PostgreSQL specifically?

Odoo only supports PostgreSQL. Several Odoo features (jsonb storage, recursive CTEs, full-text search, sequences) depend on PostgreSQL-specific behaviour. Don't try MySQL or SQLite — neither will work.

Minimum version is **PostgreSQL 12**. The Ubuntu 22.04+ package gives you 14 or 15, both fine.

## The three "users" in your head

This is the part that trips trainees up. Three different "users" are involved on Day 1, all named differently and living in different places:

| User | Lives where | What it does |
|---|---|---|
| **Your Linux user** | `/etc/passwd` (your WSL/Ubuntu login) | Owns `/opt/hcpi`, runs `odoo-bin` |
| **`postgres`** | Linux *and* PostgreSQL (created by `apt install postgresql`) | The superuser of the database server; used once to bootstrap |
| **`hcpi`** | PostgreSQL only | The database role HCPI connects as |

You're never logged in as `postgres` or `hcpi` at the Linux level — only as yourself. `sudo -u postgres ...` switches to the `postgres` Linux user just long enough to run a `psql` or `createuser` command.

## Authentication: peer vs. password

PostgreSQL on Ubuntu defaults to **peer authentication** for local connections. This means: "If the Linux user running this connection has the same name as the PostgreSQL role they're asking for, allow it. Otherwise, reject."

Concretely:

- `sudo -u postgres psql` → works, because Linux user `postgres` is asking for PostgreSQL role `postgres`. ✓
- `psql -U hcpi -d hcpi` (as your own Linux user) → fails, because you're not the Linux user `hcpi`. ✗

The fix is to add `-h localhost` to force a TCP connection, which switches PostgreSQL to **password authentication**:

```bash
psql -U hcpi -d hcpi -h localhost   # uses password
```

This is also why the HCPI configuration file has `db_host = localhost`. With `db_host = False`, Odoo uses the Unix socket and triggers peer auth — which fails because the Linux process running Odoo (you) isn't the `hcpi` Linux user (which doesn't exist). The [Configuration File](configuration.md) page covers this in detail.

!!! info "Why is the exported config wrong then?"
    On production servers, HCPI typically runs as a dedicated `hcpi` Linux user — created during deployment — so peer auth works. That's why `db_host = False` ships in the export. For training and development you're running as yourself, so you override it.

## What we create

Three commands, all run via `sudo -u postgres`:

```bash
sudo -u postgres createuser -s hcpi
sudo -u postgres psql -c "ALTER USER hcpi WITH PASSWORD 'your_secure_password';"
sudo -u postgres createdb -O hcpi hcpi
```

In order:

1. **Create a PostgreSQL role called `hcpi`.** The `-s` flag makes it a superuser within PostgreSQL. (For training that's fine; in production, Odoo only needs `CREATEDB` rather than full superuser, but `-s` keeps things simple.)
2. **Set a password.** This goes in `hcpi.conf` as `db_password` so Odoo can use it.
3. **Create a database called `hcpi`, owned by the `hcpi` role.** `-O hcpi` sets ownership.

By convention, role name and database name match. They don't have to — but it keeps things simple, and the rest of the docs assume it.

## Verifying the setup

Before moving on:

```bash
psql -U hcpi -d hcpi -h localhost -c "SELECT version();"
```

You should be prompted for the password (the one you set in step 2) and then see something like:

```
                                                   version
---------------------------------------------------------------------------------------------------------
 PostgreSQL 14.x on x86_64-pc-linux-gnu, compiled by gcc ...
```

If you get `FATAL: password authentication failed` — your password doesn't match. Re-run the `ALTER USER` command with the right password.

If you get `could not connect to server: No such file or directory` — PostgreSQL isn't running. Start it:

```bash
sudo systemctl start postgresql     # systemd-enabled systems
sudo service postgresql start       # WSL1 or older
```

## Common operations you'll want later

You won't need these on Day 1, but you'll come back to them.

### Reset the `hcpi` user's password

If you forget it or want to rotate:

```bash
sudo -u postgres psql -c "ALTER USER hcpi WITH PASSWORD 'new_password';"
```

Then update `db_password` in `hcpi.conf` to match.

### Drop and recreate the database

When you want to start over (failed restore, want an empty DB):

```bash
# Make sure Odoo isn't running first — Ctrl+C in its terminal
sudo -u postgres dropdb hcpi
sudo -u postgres createdb -O hcpi hcpi
```

If `dropdb` complains the DB is in use, kill open connections:

```bash
sudo -u postgres psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='hcpi' AND pid <> pg_backend_pid();"
sudo -u postgres dropdb hcpi
```

### List databases / roles

```bash
sudo -u postgres psql -c "\l"      # list databases
sudo -u postgres psql -c "\du"     # list roles
```

### Connect for ad-hoc queries

```bash
psql -U hcpi -d hcpi -h localhost
# At the hcpi=# prompt:
\dt                                # list tables
SELECT count(*) FROM res_users;    # any SQL works
\q                                 # quit
```

## Practical steps

- **[Windows WSL](../../installation/windows-wsl.md)** — Step 5 (PostgreSQL configure, including the systemd nuance)
- **[Linux Server](../../installation/linux.md)** — Step 3

➡️ Next: [Virtual Environment & HCPI Install](virtualenv-install.md)
