---
name: secrets
description: |
  MUST be invoked before any work involving: ov secrets commands, KeePass .kdbx credential management, credential import/export, or secret database administration.
---

# Secrets - KeePass Credential Management

## Overview

Manage credentials in a KeePass `.kdbx` database. Part of ov's credential store hierarchy — the kdbx backend is used when the system keyring is unavailable (headless servers, SSH sessions) or when explicitly selected via `ov settings set secret_backend kdbx`.

## Credential Store Hierarchy

Resolution order (first match wins):

| Priority | Backend | When used |
|----------|---------|-----------|
| 1 | Environment variable | `OV_VNC_PASSWORD`, etc. |
| 2 | System keyring | GNOME Keyring, KDE Wallet, KeePassXC |
| 3 | KeePass .kdbx | Headless/SSH or `secret_backend: kdbx` |
| 4 | Config file | Plaintext fallback in `config.yml` |

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Create database | `ov secrets init [path]` | Create new .kdbx, auto-set config |
| List entries | `ov secrets list [service]` | List entries, optionally filter by service |
| Get value | `ov secrets get <service> <key>` | Print credential value |
| Set value | `ov secrets set <service> <key> [value]` | Set credential (prompts if no value) |
| Generate value | `ov secrets set <service> <key> --generate` | Generate 32-char hex random value |
| Delete entry | `ov secrets delete <service> <key>` | Remove an entry |
| Import | `ov secrets import [--dry-run]` | Import from config.yml + keyring |
| Export | `ov secrets export [--format yaml\|json]` | Export all entries (plaintext!) |
| Show path | `ov secrets path` | Print resolved .kdbx file path |

## Subcommand Details

### Init

```bash
ov secrets init                          # Default: ~/.config/ov/secrets.kdbx
ov secrets init /path/to/secrets.kdbx    # Custom path
ov secrets init --force                  # Overwrite existing database
```

Creates a new .kdbx database with password confirmation. Automatically sets `secrets.kdbx_path` in config. After init, activate with:

```bash
ov settings set secret_backend kdbx
```

Or it auto-activates when the system keyring is unavailable.

### List

```bash
ov secrets list              # All ov entries
ov secrets list ov/vnc       # Filter by service prefix
```

Prints `service/key` pairs, one per line.

### Get / Set / Delete

```bash
ov secrets get ov/vnc my-image              # Print value
ov secrets set ov/vnc my-image              # Prompt for value securely
ov secrets set ov/vnc my-image mypassword   # Set value inline
ov secrets set ov/vnc my-image --generate   # Generate and print random value
ov secrets delete ov/vnc my-image           # Remove entry
```

The `--generate` flag produces 16 random bytes as 32-character hex string, printed to stdout.

### Import

```bash
ov secrets import --dry-run    # Preview what would be imported
ov secrets import              # Import from config.yml + keyring
```

Collects credentials from:
1. **Config file** — plaintext credentials in `config.yml`
2. **System keyring** — entries tracked in `keyring_keys` config list

Shows source and success/failure for each entry.

### Export

```bash
ov secrets export                  # YAML format (default)
ov secrets export --format json    # JSON format
```

**WARNING:** Exports plaintext credentials. Outputs nested map: `service -> key -> value`.

### Path

```bash
ov secrets path    # Prints resolved path or default location
```

## Common Workflows

### First-Time Setup

```bash
ov secrets init                         # Create database
ov settings set secret_backend kdbx     # Activate kdbx backend
ov secrets import --dry-run             # Preview migration
ov secrets import                       # Migrate existing credentials
```

### Backup / Restore

The `.kdbx` file is a standard KeePass database. You can open it with KeePassXC or any KeePass-compatible tool for backup or manual editing.

## Password Caching

The kdbx master password is cached in the Linux kernel keyring (user keyring, key `ov-kdbx-password`) after the first interactive prompt. All subsequent `ov` commands within the timeout window reuse the cached password automatically -- no re-prompting.

**Password resolution chain:**
1. `OV_KDBX_PASSWORD` environment variable (CI/automation)
2. Linux kernel keyring lookup
3. Interactive prompt (systemd-ask-password or terminal)
4. After prompting, auto-store in kernel keyring with configured TTL

**Configuration:**

```bash
ov settings set secrets.kdbx_cache false       # Disable caching entirely
ov settings set secrets.kdbx_cache_timeout 7200 # Cache for 2 hours instead of 1
```

Default: enabled, 3600 seconds (1 hour) TTL. Uses `keyctl` syscalls via `golang.org/x/sys/unix`. Source: `ov/keyctl.go`.

## Config Keys

| Key | Description |
|-----|-------------|
| `secret_backend` | Force backend: `keyring`, `kdbx`, or `config` |
| `secrets.kdbx_path` | Path to .kdbx file |
| `secrets.kdbx_key_file` | Optional key file for .kdbx |
| `secrets.kdbx_cache` | Enable/disable kernel keyring caching (default: `true`) |
| `secrets.kdbx_cache_timeout` | TTL in seconds for cached password (default: `3600`) |

## Project-Level Secrets (.secrets + direnv)

Separate from ov's credential store, project-level environment variables (e.g., `GMAIL_USER`, `GMAIL_PASSWORD`) are stored in `.secrets` — a GPG-encrypted file at the project root. direnv decrypts it in memory via `dotenv_gpg_if_exists` when entering the directory.

**This is NOT managed by `ov secrets` (kdbx).** The two systems serve different purposes:

| System | What it manages | How it works |
|--------|----------------|--------------|
| `ov secrets` (kdbx/keyring) | Container-level credentials (VNC passwords, service secrets) | Provisioned at `ov config` time into Podman secrets |
| `ov secrets gpg` + direnv | Project-level shell env vars (API keys, credentials) | GPG-encrypted `.secrets` file, decrypted by direnv on `cd` |

## `ov secrets gpg` — GPG-Encrypted .secrets Management

Manage GPG-encrypted `.secrets` environment files directly from the CLI. All commands shell out to `gpg` (must be in PATH).

### Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Show contents | `ov secrets gpg show [-f FILE]` | Decrypt and print to stdout |
| Edit in editor | `ov secrets gpg edit [-f FILE]` | Decrypt, open `$EDITOR`, re-encrypt |
| Encrypt file | `ov secrets gpg encrypt -r KEY_ID [-i .env] [-o .secrets]` | Encrypt plaintext env file |
| Decrypt file | `ov secrets gpg decrypt [-i .secrets] [-o FILE]` | Decrypt to file or stdout |
| Set a key | `ov secrets gpg set KEY VALUE [-f FILE] [-r KEY_ID]` | Add or update KEY=VALUE |
| Remove a key | `ov secrets gpg unset KEY [-f FILE]` | Remove a key from .secrets |
| Add recipient | `ov secrets gpg add-recipient KEY_ID [-f FILE]` | Re-encrypt with additional recipient |
| List recipients | `ov secrets gpg recipients [-f FILE]` | List GPG key IDs that can decrypt |

### Common Workflows

```bash
# Create .secrets from a plaintext .env file
ov secrets gpg encrypt -r 420DE2B3 -i .env -o .secrets
rm .env

# View contents
ov secrets gpg show

# Add a new secret
ov secrets gpg set API_KEY sk-test-abc123

# Edit in your editor
ov secrets gpg edit

# Remove a secret
ov secrets gpg unset OLD_KEY

# Add another person who can decrypt
ov secrets gpg add-recipient THEIR_KEY_ID

# See who can decrypt
ov secrets gpg recipients
```

### Flags

- `-f, --file` — Path to encrypted file (default: `.secrets` in current directory)
- `-r, --recipient` — GPG key ID (repeatable, required for `encrypt`, optional for `set` on new files)
- `-i, --input` — Input file for encrypt/decrypt (default: `.env` / `.secrets`)
- `-o, --output` — Output file for encrypt/decrypt (default: `.secrets` / stdout)

### Using .secrets Inside Containers

For GPG agent forwarding into containers (so `gpg --decrypt` works inside), use the `agent-forwarding` layer. See `/ov-layers:agent-forwarding` for details. The container's GPG uses the host's agent via a forwarded socket — no GPG agent runs inside the container.

## Cross-References

- `/ov:config` — `secret_backend`, `secrets.kdbx_path` settings keys
- `/ov:service` — container secrets (`secrets` field in layer.yml, provisioned at `ov config`)
- `/ov-layers:agent-forwarding` — SSH/GPG agent forwarding into containers
- `/ov-layers:gnupg` — GnuPG package layer
- `/ov-layers:direnv` — direnv environment loader
- `gpg-agent-setup/CLAUDE.md` — GPG agent + direnv setup for `.secrets` files

## Source

`ov/secrets_cmd.go` (CLI commands), `ov/secrets_gpg.go` (GPG .secrets commands), `ov/credential_kdbx.go` (KdbxStore backend).

## When to Use This Skill

**MUST be invoked** when the task involves `ov secrets` commands, KeePass .kdbx credential management, GPG-encrypted `.secrets` file management, credential import/export, or secret database administration. Invoke this skill BEFORE reading source code or launching Explore agents.
