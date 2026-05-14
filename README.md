# manage_user

An Ansible role for creating and managing user accounts and service accounts on Ubuntu systems. Accounts are configured for SSH key authentication only — password login is explicitly locked. Sudo access is optional and granted without a password prompt via a dedicated sudoers drop-in file.

Users and service accounts are declared as lists in separate var files, making it straightforward to manage a full set of accounts in a single playbook run.

---

## Requirements

- **Control node:** Ansible 2.14+
- **Target node:** Ubuntu 18.04 (Bionic), 20.04 (Focal), 22.04 (Jammy), or 24.04 (Noble)
- **Python on target:** Python 3 must be available at `/usr/bin/python3` (standard on all supported Ubuntu releases)
- **Privilege escalation:** The role must run with `become: true` as it modifies system accounts and `/etc/sudoers.d/`
- **Collections:** Install before first use:

```bash
ansible-galaxy collection install -r requirements.yml
```

| Collection | Minimum version | Used for |
|---|---|---|
| `ansible.posix` | 1.5.0 | `authorized_key` module |
| `community.general` | 7.0.0 | General Ubuntu compatibility |

---

## Role Variables

All variables are defined in `defaults/main.yml`.

### Global defaults

These apply to every entry in `manage_users` and `manage_user_service_accounts` unless overridden on the individual item.

| Variable | Default | Description |
|---|---|---|
| `manage_user_default_home_base` | `/home` | Base path for home directories. The account home is `<base>/<name>` unless `home` is set on the item. |
| `manage_user_default_shell` | `/bin/bash` | Login shell for regular user accounts. |
| `manage_user_service_account_default_shell` | `/usr/sbin/nologin` | Login shell for service accounts. |

### `manage_user_accounts`

A list of interactive user accounts. Each item supports:

| Key | Required | Default | Description |
|---|---|---|---|
| `name` | yes | — | Username |
| `ssh_public_key` | yes | — | Full public key string (e.g. `ssh-ed25519 AAAA… user@host`) |
| `state` | no | `present` | `present` to create/update, `absent` to remove |
| `uid` | no | auto-assigned | Fixed UID for the account. Useful for consistency across hosts. |
| `comment` | no | `""` | GECOS field — typically the user's full name |
| `sudo` | no | `false` | Grant full `NOPASSWD` sudo via `/etc/sudoers.d/` |
| `groups` | no | `[]` | Additional groups to assign |
| `shell` | no | `manage_user_default_shell` | Override the login shell for this user |
| `home` | no | `manage_user_default_home_base/<name>` | Override the home directory path |
| `ssh_key_exclusive` | no | `false` | When `true`, removes any authorized keys not declared here |
| `remove_home` | no | `false` | When `state: absent`, also delete the home directory |

### `manage_user_service_accounts`

A list of non-interactive service accounts. No SSH key is required. Accounts are created as system accounts (low UID) with a nologin shell.

| Key | Required | Default | Description |
|---|---|---|---|
| `name` | yes | — | Account name |
| `state` | no | `present` | `present` to create/update, `absent` to remove |
| `uid` | no | auto-assigned | Fixed UID for the account. Use values below 1000 for system accounts. |
| `comment` | no | `""` | GECOS field — descriptive label for the account |
| `groups` | no | `[]` | Additional groups to assign |
| `shell` | no | `manage_user_service_account_default_shell` | Override the shell for this account |
| `home` | no | `manage_user_default_home_base/<name>` | Override the home directory path |
| `sudo` | no | `false` | Grant full `NOPASSWD` sudo via `/etc/sudoers.d/` |
| `remove_home` | no | `false` | When `state: absent`, also delete the home directory |

---

## Defining Users and Service Accounts

Accounts are declared in two var files loaded by the playbook. Both are gitignored — create them before running:

**`vars/users.yml`**

```yaml
manage_user_accounts:
  - name: alice
    uid: 1001
    comment: "Alice Smith"
    ssh_public_key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... alice@example.com"
    sudo: true

  - name: bob
    uid: 1002
    comment: "Bob Jones"
    ssh_public_key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... bob@example.com"
    groups:
      - docker

  - name: carol
    uid: 1003
    comment: "Carol White"
    ssh_public_key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... carol@example.com"
    sudo: true
    groups:
      - docker
      - adm
    ssh_key_exclusive: true

  - name: dave
    comment: "Dave Brown"
    ssh_public_key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... dave@example.com"
    state: absent
    remove_home: true
```

**`vars/service_accounts.yml`**

```yaml
manage_user_service_accounts:
  - name: deploy
    uid: 900
    comment: "Deployment automation"
    groups:
      - www-data

  - name: monitoring
    uid: 901
    comment: "Metrics and alerting"
    home: /var/lib/monitoring

  - name: backup
    uid: 902
    comment: "Backup agent"
    home: /var/lib/backup
    groups:
      - disk

  - name: ci-runner
    comment: "CI pipeline runner"
    groups:
      - docker
    state: absent
    remove_home: true
```

---

## How the Role Works

### Task file structure

The role's tasks are split into discrete files:

| File | Responsibility |
|---|---|
| `tasks/main.yml` | Loops over `manage_user_accounts` and `manage_user_service_accounts`, passing resolved vars to `manage_user.yml` for each entry |
| `tasks/manage_user.yml` | Per-entry wrapper — sequences validate, user, ssh, and sudo |
| `tasks/validate.yml` | Asserts required fields are present |
| `tasks/user.yml` | Ensures supplementary groups exist, then creates or removes the account via the `user` module |
| `tasks/ssh.yml` | Configures the authorized key (skipped for service accounts) |
| `tasks/sudo.yml` | Grants or removes `NOPASSWD` sudo access |

### Creating or updating an account (`state: present`)

1. **Validation** — asserts `name` is non-empty; for regular users, also asserts `ssh_public_key` is provided.
2. **Supplementary groups** — any group listed under `groups` that does not already exist on the host is created before the account is configured. This allows groups like `docker` to be declared even when the software that would normally create them (e.g. Docker) has not yet been installed.
3. **User account** — creates or reconciles the account with the declared shell, home, comment, and groups. Password login is locked (`password_lock: true`). Service accounts are created as system accounts (`system: true`).
4. **SSH authorized key** — writes the public key to `<home>/.ssh/authorized_keys` using an explicit path derived from `manage_user_home`. Skipped entirely for service accounts. If `ssh_key_exclusive: true`, all other keys in the file are removed. Using an explicit path means this task behaves correctly in `--check` mode even when the account does not yet exist on the target host.
5. **Sudo access** — if `sudo: true`, writes `/etc/sudoers.d/<name>` validated with `visudo -cf` before placement. If `sudo: false`, removes any existing sudoers entry for the account.

### Removing an account (`state: absent`)

1. Removes the account. If `remove_home: true`, also deletes the home directory and mail spool.
2. Removes `/etc/sudoers.d/<name>` if present.

### Idempotency

All tasks are idempotent. Running the role repeatedly against the same host produces no changes when the system already matches the declared state.

---

## Running the Playbook

```bash
# Install required collections (first time only)
ansible-galaxy collection install -r requirements.yml

# Run against all hosts
ansible-playbook -i inventory.ini playbook.yml

# Limit to a specific host or group
ansible-playbook -i inventory.ini playbook.yml --limit server1

# Dry run — no changes made
ansible-playbook -i inventory.ini playbook.yml --check

# Verbose output
ansible-playbook -i inventory.ini playbook.yml -v
```

---

## Secret Scanning

This repository uses [gitleaks](https://github.com/gitleaks/gitleaks) to detect accidentally committed secrets.

### Pre-commit hook

A pre-commit hook runs `gitleaks protect --staged` before every commit, blocking the commit if a secret is detected in the staged diff. The hook is managed via [pre-commit](https://pre-commit.com) and configured in `.pre-commit-config.yaml`. Install it once after cloning:

```bash
pip install pre-commit   # or: brew install pre-commit
pre-commit install
```

gitleaks is downloaded automatically by pre-commit — no separate installation needed.

### GitHub Actions

Two workflows run on every push and pull request:

| Workflow | Purpose |
|---|---|
| `.github/workflows/ansible-lint.yml` | Lints all Ansible content at the `production` profile |
| `.github/workflows/gitleaks.yml` | Scans the full commit history for secrets; uploads findings to **Security → Code scanning** as SARIF alerts |

The gitleaks workflow installs gitleaks directly from the GitHub releases API so no licence secret is required.

### Configuration

Rules are defined in `.gitleaks.toml`, which extends the default gitleaks ruleset.

---

## Security Notes

- **Password login is always locked.** The `user` module is called with `password_lock: true`, setting the password field to `!` in `/etc/shadow`. Even if SSH password authentication is enabled at the sshd level, these accounts cannot authenticate with a password.
- **Sudoers validation.** The sudoers drop-in is validated with `visudo -cf` before being written. If the file is malformed, Ansible aborts rather than writing a broken sudoers configuration.
- **Sudoers cleanup is explicit.** Setting `sudo: false` on a subsequent run actively removes `/etc/sudoers.d/<name>`. Sudo access is never silently left in place.
- **SSH key exclusivity.** By default the role appends the declared key without touching others. Set `ssh_key_exclusive: true` on an account if it should only trust the declared key.
- **Service accounts cannot log in interactively.** The default shell is `/usr/sbin/nologin` and no SSH key is configured, preventing both shell and key-based access.
