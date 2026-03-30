---
name: enc
description: |
  MUST be invoked before any work involving: encrypted storage, gocryptfs, or encrypted bind mounts.
---

# Enc - Encrypted Storage

## Overview

Encrypted bind mount management is integrated into the `ov config` command. Gocryptfs-encrypted volumes store sensitive data (credentials, keys, configs) with transparent encryption at rest. The cipher directory lives on disk; the plain directory is mounted on demand. `ov config <image>` handles initialization and mounting during deployment setup. `ov start` mounts encrypted volumes inline before starting the container.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Setup (init + mount) | `ov config <image>` | Initialize cipher dirs and mount encrypted volumes |
| Mount | `ov config mount <image>` | Mount encrypted volumes |
| Unmount | `ov config unmount <image>` | Unmount encrypted volumes |
| Status | `ov config status <image>` | Show mount status |
| Change password | `ov config passwd <image>` | Change encryption password |

All commands accept `--volume NAME` to target a specific volume (otherwise all encrypted volumes are affected).

## Configuration

Encrypted bind mounts are declared in `images.yml` or `deploy.yml`:

```yaml
bind_mounts:
  - name: secrets
    path: "~/.myapp/secrets"    # Container mount path
    encrypted: true             # gocryptfs-managed
```

Rules:
- `encrypted: true`: `host` field must be empty -- ov manages cipher/plain directories
- `path`: container mount path (required). `~`/`$HOME` expanded to the image's resolved home
- `name`: unique identifier, must match `^[a-z0-9]+(-[a-z0-9]+)*$`
- Names must not collide with layer volume names (same namespace)

## Storage Layout

```
~/.local/share/ov/encrypted/
  ov-<image>-<name>/
    cipher/    # Encrypted data (always on disk)
    plain/     # Decrypted mount point (mounted on demand)
```

Override base path: `ov settings set encrypted_storage_path /path/to/storage` or `OV_ENCRYPTED_STORAGE_PATH=/path`.

## Commands

### Setup (ov config)

```bash
ov config my-app                             # Init + mount all encrypted volumes
ov config my-app --password auto             # Auto-generate password
ov config my-app --password manual           # Prompt for password
ov config my-app --volume secrets            # Target specific volume
```

`ov config <image>` handles both initialization (creating cipher directories) and mounting in a single step. If volumes are already initialized, it mounts them. Password is cached in kernel keyring for multi-volume images.

### Mount

```bash
ov config mount my-app                    # Mount all encrypted volumes
ov config mount my-app --volume secrets   # Mount specific volume
```

Prompts for password (or reuses from keyring). The plain directory becomes available for container bind mounts.

### Unmount

```bash
ov config unmount my-app                    # Unmount all
ov config unmount my-app --volume secrets   # Unmount specific
```

### Status

```bash
ov config status my-app
# secrets: mounted
# configs: not mounted
```

### Change Password

```bash
ov config passwd my-app
```

Changes the gocryptfs password for all encrypted volumes of an image.

## Single Password

When an image has multiple encrypted bind mounts, `ov config`, `ov config mount`, and the generated crypto systemd unit all use `systemd-ask-password --id=ov-<image>` to cache the passphrase in the kernel keyring. Password is prompted once and reused for all volumes.

## KeePass Integration

Use the `--kdbx` global flag to specify a KeePass database for password storage:

```bash
ov --kdbx ~/.config/ov/secrets.kdbx config my-app
```

## Integration with Runtime

- **`ov shell`/`ov start` (direct mode)**: resolves bind mounts, verifies encrypted volumes are mounted, appends `-v <plain>:<container-path>` flags. `ov start` mounts encrypted volumes inline before starting the container
- **`ov start --enable`**: generates quadlet file and starts the service. Encrypted volumes are mounted inline by `ov start` -- no companion service needed
- **`ov start --enable=false`**: starts without generating a quadlet file
- **`ov remove`**: `--purge` removes named volumes
- **`ov seed <image>`**: copies default data from the image into empty bind mount directories (works for both plain and encrypted mounts after mounting)
- **`ov inspect --format bind_mounts`**: outputs `NAME\tHOST\tPATH\tENCRYPTED`

## Bind Mount / Volume Override

When a bind mount has the same name as a layer volume, the bind mount **overrides** the volume. The named volume is not created -- the bind mount is used instead. This allows deploy.yml to replace layer-declared volumes with encrypted bind mounts.

```yaml
# layer.yml declares a volume:
volumes:
  - name: data
    path: "~/.myapp"

# deploy.yml overrides with encrypted bind mount:
bind_mounts:
  - name: data          # Same name as layer volume
    path: "~/.myapp"
    encrypted: true
```

## Plain (Non-Encrypted) Bind Mounts

For comparison, plain bind mounts require an explicit `host` path:

```yaml
bind_mounts:
  - name: data
    host: "~/data/myapp"       # Host directory (required)
    path: "~/.myapp"           # Container path
```

Plain mounts do not use encrypted storage commands. They are direct bind mounts.

Source: `ov/enc.go`, `ov/validate.go` (`validateBindMounts`).

## Cross-References

- `/ov:deploy` -- Quadlet integration, bind mount configuration, deploy.yml
- `/ov:config` -- `encrypted_storage_path` setting
- `/ov:service` -- Container lifecycle, `ov start` inline mount

## When to Use This Skill

**MUST be invoked** when the task involves encrypted storage, gocryptfs, or encrypted bind mounts. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Pre-deployment. Set up encrypted storage before `ov start`. See also `/ov:deploy` (bind mounts).
