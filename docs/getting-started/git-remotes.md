# Adding the EAC Repository as a Remote

If you got the HCPI code from a country server export, a colleague's archive, or any source other than a direct clone of the EAC repo, the git history inside `custom/HCPI/` may point at a remote you **don't have push rights to** — or have no usable remote at all.

To contribute changes back, you'll need to add the canonical **East African Community HCPI repo** as a remote. That's where you have permission to push.

Upstream URL: **<https://github.com/East-African-Community-HCPI/HCPI>**

!!! info "When you don't need this page"
    If you used [Option 1 (clone the EAC upstream repo)](getting-the-code.md#option-1-clone-the-eac-upstream-repo) on the "Getting the Code" page, `origin` already points at the EAC repo. You can skip this page entirely.

## Step 1: Check what you currently have

Move into the HCPI module folder and look at the current remotes:

```bash
cd /opt/hcpi/custom/HCPI
git remote -v
```

You'll see one of three things:

- **Nothing** — the folder isn't a git repo, or has no remotes configured.
- **An `origin` you can't push to** — typically a country server's bare repo, or a repo where you weren't granted write access.
- **`origin` already points at `East-African-Community-HCPI/HCPI`** — you're done; nothing to add.

## Step 2: Add the EAC repo as a remote

Pick **one** of the two URL styles. SSH is recommended for day-to-day work; HTTPS is fine if you haven't set up SSH keys yet.

=== "SSH (recommended)"

    Requires an SSH key registered with your GitHub account. See [SSH key setup](#ssh-key-setup) below if you don't have one yet.

    ```bash
    git remote add eac git@github.com:East-African-Community-HCPI/HCPI.git
    ```

=== "HTTPS"

    Works without SSH setup. Git will prompt for your GitHub username and a **Personal Access Token** (not your password) the first time you push. Generate one at <https://github.com/settings/tokens> with the `repo` scope.

    ```bash
    git remote add eac https://github.com/East-African-Community-HCPI/HCPI.git
    ```

The name `eac` is just a label — you can call it anything (`upstream`, `eachq`, etc.). `eac` is short and clear, so the rest of this page uses it.

## Step 3: Verify and fetch

```bash
git remote -v
# origin  <whatever it was>            (fetch)
# origin  <whatever it was>            (push)
# eac     git@github.com:East-African-Community-HCPI/HCPI.git (fetch)
# eac     git@github.com:East-African-Community-HCPI/HCPI.git (push)

git fetch eac
```

The fetch downloads all branches from the EAC repo into `eac/*` refs. You should see the country branches listed in [Country Variants](../understanding-the-codebase/country-variants.md) — `eac/18.0`, `eac/ug_18`, `eac/ke_18_mtnce`, etc.

## Step 4: Push your changes

Create a branch for your work and push it to `eac`:

```bash
git checkout -b my-feature-branch
# ... make your edits, commit ...
git push -u eac my-feature-branch
```

The `-u` flag sets `eac/my-feature-branch` as the upstream for this branch, so future `git push` and `git pull` calls don't need the remote name.

Then open a pull request on GitHub against the appropriate base branch (usually `18.0` or your country branch).

## Optional: replace origin instead of adding a second remote

If your existing `origin` is useless to you (e.g. a country server path you'll never push back to), it's cleaner to replace it outright:

```bash
git remote set-url origin git@github.com:East-African-Community-HCPI/HCPI.git
git fetch origin
```

Or, if you want to keep the old one around for reference, rename it first:

```bash
git remote rename origin country-server
git remote add origin git@github.com:East-African-Community-HCPI/HCPI.git
git fetch origin
```

Either way, `origin` is now the EAC repo and you don't need the separate `eac` name.

## SSH key setup

If you chose the SSH URL but haven't set up a key on this machine yet:

```bash
# 1. Generate a key (press Enter through all prompts for defaults)
ssh-keygen -t ed25519 -C "your-email@example.com"

# 2. Print the public key
cat ~/.ssh/id_ed25519.pub
```

Copy the output (starts with `ssh-ed25519 ...`) and paste it at <https://github.com/settings/keys> → **New SSH key**.

Test the connection:

```bash
ssh -T git@github.com
# Hi <your-username>! You've successfully authenticated...
```

On Windows native (no WSL), run the same commands inside Git Bash. On WSL, run them inside the WSL shell — the key lives in the WSL filesystem, separate from any Windows-side key.

## Troubleshooting

**`Permission denied (publickey)` on SSH push** — your SSH key isn't registered with GitHub, or you're using a different key than the one you added. Run `ssh -T git@github.com` to confirm which account GitHub sees.

**`remote: Permission to East-African-Community-HCPI/HCPI.git denied`** — you're authenticated, but your GitHub account hasn't been granted write access to the EAC organisation. Request access from [mkakinyi@eachq.org](mailto:mkakinyi@eachq.org).

**HTTPS keeps asking for credentials** — install a credential helper:

```bash
# Linux
git config --global credential.helper store

# Windows / WSL
git config --global credential.helper manager
```

Then push once more and your token will be cached.
