# User Administration

Once HCPI is running, you need to be able to log in, add other users, reset passwords when people forget them, and turn on Developer Mode (because you can't navigate the codebase without it). This page is the practical reference for all of that.

## First login

Open your browser to `http://localhost:9201`.

**If you restored from a real database** (Option A in the install guide): log in with whatever credentials work on the source instance — the users are already in your DB.

**If you started with an empty database** (Option B): Odoo creates one default user.

- **Login**: `admin`
- **Password**: `admin`

Change the password immediately:

1. Top-right user menu → **My Profile**
2. **Preferences** tab → **Change Password**

Then create a second admin-level user so you don't lock yourself out if the first one breaks (covered below).

## Two passwords that get confused

Trainees mix these up a lot:

| Password | Used for | Lives where |
|---|---|---|
| **User login password** (e.g. admin's) | Logging into HCPI through the web UI | Stored hashed in the `res_users` table in PostgreSQL |
| **Master password** (`admin_passwd`) | Database-level ops via Odoo's web DB manager: create, duplicate, drop, backup, restore | Plain text in `/opt/hcpi/conf/hcpi.conf` |

They are *completely independent*. Resetting one does nothing to the other.

## Activating Developer Mode

You can't navigate the code or do most admin work without this. Turn it on once per session (it persists until logout):

1. Top-right user menu → **Settings**
2. Inside the Settings app, scroll down — you'll see "Developer Tools" with a link **Activate the developer mode**
3. Click it. The page reloads with a bug 🐛 icon in the top bar.

Now hovering any field shows its technical name (model + field). The browser URL changes to expose record IDs and view IDs. You also get a "Developer" menu (under the bug icon) with "Edit View", "Edit Action", etc. — vital for finding the file that produces a piece of UI.

!!! tip "URL trick"
    You can also activate developer mode by appending `?debug=1` (or `?debug=assets` for full asset reload) to any HCPI URL. Useful when the Settings page itself is broken.

## Adding a new user

1. **Settings** → **Users & Companies** → **Users**
2. Click **New**
3. Fill in:
    - **Name** — full name (used in the UI everywhere)
    - **Login** — what they type to sign in (usually an email)
    - **Email** — used for password-reset emails (if SMTP is configured)
4. **Access Rights** tab — pick the modules and the role (`User: All Documents`, `Administrator`, etc.)
5. **Save**
6. To send them a setup link instead of choosing a password yourself, click **Action → Send Password Reset Instructions** at the top of the form. (Requires `email_from` and SMTP settings in `hcpi.conf`. For training without SMTP, set the password manually instead — see the next section.)

### Setting a password manually (no SMTP available)

In training, SMTP is usually not wired up. So instead of sending a reset link:

1. Save the new user
2. Log out, then on the login page click **Reset Password**, enter their login, and intercept the link from the log file (Odoo logs the reset link to `/opt/hcpi/log/hcpi.log` when no SMTP is configured), or

A simpler approach: as admin, with developer mode on, go to the user's record, click the **Action** menu → **Change Password**, and set one directly. Tell the user, they log in, then immediately change it from My Profile.

## Resetting a user's password (you're the admin)

When somebody forgets theirs and you're logged in as an admin:

1. **Settings** → **Users & Companies** → **Users**
2. Open the user's record
3. **Action** menu (top-left of the form) → **Change Password**
4. Enter a new password, save
5. Tell them, ask them to change it on first login

This is the most common reset and what you'll do day-to-day.

## Resetting the admin password (you're locked out)

If nobody can log in as admin — fresh recovery scenario — you do this directly in PostgreSQL.

!!! warning "Stop Odoo first"
    Editing `res_users` while Odoo is running can cause sessions/cache to disagree with the database. Stop Odoo (`Ctrl+C`) before running these commands.

**Option 1: Set a known plain password (simplest, training-friendly)**

Run this with Odoo *stopped*:

```bash
sudo -u postgres psql -d hcpi -c \
  "UPDATE res_users SET password = 'admin' WHERE login = 'admin';"
```

Then start Odoo and log in as `admin` / `admin`. Odoo will detect the plaintext password on first authentication, validate it, and silently rehash it. Change the password immediately from My Profile after logging in.

If the `admin` login was renamed in your DB, find the right one:

```bash
sudo -u postgres psql -d hcpi -c "SELECT id, login FROM res_users;"
```

**Option 2: Hash a password and write it directly**

If you'd rather not rely on Odoo's plaintext-detection behaviour, hash it yourself using Odoo's own `passlib` config. With the venv active:

```bash
cd /opt/hcpi
source venv/bin/activate
python -c "from passlib.context import CryptContext; print(CryptContext(['pbkdf2_sha512']).hash('your_new_password'))"
```

Copy the hash it prints (it'll start with `$pbkdf2-sha512$...`). Then:

```bash
sudo -u postgres psql -d hcpi -c \
  "UPDATE res_users SET password = '<paste_hash_here>' WHERE login = 'admin';"
```

Start Odoo, log in with `your_new_password`.

## Changing the master password (`admin_passwd`)

This isn't done in the UI — it's a config-file edit.

1. Stop Odoo
2. Open `/opt/hcpi/conf/hcpi.conf`
3. Find `admin_passwd = ...` and replace the value
4. Save and restart Odoo

That's it. The web DB manager will accept the new password from now on.

!!! info "Storage of admin_passwd"
    Older Odoo versions stored this in plain text in the config file. Odoo 16+ supports a hashed form — Odoo will rewrite the line as a hash on first use if the value is plain text. Either way, it's read at startup from the file.

## Disabling or deleting a user

Don't delete `res_users` records — too many foreign keys point at them (audit fields like `create_uid`, `write_uid`). Instead:

1. Open the user's record (Settings → Users & Companies → Users)
2. Click the **Archive** action (or set the `Active` field to false in developer mode)

Archived users can't log in but their historical record references are preserved.

## Internal Users vs. Portal Users vs. Public

Odoo has three categories of user. In HCPI:

| Type | What they can do | When you'd create one |
|---|---|---|
| **Internal User** | Full access to HCPI's web UI based on assigned groups | Your enumerators, statisticians, admins |
| **Portal User** | Limited self-service access (typical Odoo use case: customer portals) | Rarely used in HCPI |
| **Public** | The unauthenticated visitor — no DB session | Not relevant for HCPI |

For HCPI everyone gets created as an Internal User. Access within HCPI is then controlled by **groups** assigned on the user form's Access Rights tab — Day 7 covers this in detail.

## Quick reference card

| Task | How |
|---|---|
| Activate Developer Mode | Settings → Activate the developer mode (or `?debug=1` in URL) |
| Find a field's technical name | Hover the field with developer mode on |
| Add a user | Settings → Users & Companies → Users → New |
| Reset another user's password (admin) | Open user → Action → Change Password |
| Reset admin password (locked out) | `UPDATE res_users SET password = '...' WHERE login = 'admin';` from psql, stopped Odoo |
| Change master password | Edit `admin_passwd` in `hcpi.conf`, restart |
| Disable a user | Open user → Archive |

## Practical steps

There's no separate install-guide section for this — it's all on this page. Verify by:

1. Logging in to `http://localhost:9201`
2. Activating developer mode
3. Creating one extra test user and resetting their password

Day 1 complete. ✅

➡️ Continue to [Understanding the Codebase](../../understanding-the-codebase/index.md) when you're ready to start looking at the code itself.
