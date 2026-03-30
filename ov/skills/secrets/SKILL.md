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

## Config Keys

| Key | Description |
|-----|-------------|
| `secret_backend` | Force backend: `keyring`, `kdbx`, or `config` |
| `secrets.kdbx_path` | Path to .kdbx file |
| `secrets.kdbx_key_file` | Optional key file for .kdbx |

## Cross-References

- `/ov:config` — `secret_backend`, `secrets.kdbx_path` settings keys
- `/ov:service` — container secrets (`secrets` field in layer.yml, provisioned at `ov config`)

## Source

`ov/secrets_cmd.go` (CLI commands), `ov/credential_kdbx.go` (KdbxStore backend).

## When to Use This Skill

**MUST be invoked** when the task involves the `ov secrets` command, KeePass credential management, or the kdbx credential backend. Invoke this skill BEFORE reading source code or launching Explore agents.
