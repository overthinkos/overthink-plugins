---
name: crypto
description: |
  Encrypted storage: ov crypto init/mount/unmount/status/passwd commands.
  Use when working with gocryptfs encrypted bind mounts.
---

# Crypto - Encrypted Storage

## Overview

`ov crypto` manages gocryptfs-encrypted bind mounts for container images. Encrypted volumes store sensitive data (credentials, keys, configs) with transparent encryption at rest. The cipher directory lives on disk; the plain directory is mounted on demand.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Initialize | `ov crypto init <image>` | Create gocryptfs cipher directories |
| Mount | `ov crypto mount <image>` | Mount encrypted volumes |
| Unmount | `ov crypto unmount <image>` | Unmount encrypted volumes |
| Status | `ov crypto status <image>` | Show mount status |
| Change password | `ov crypto passwd <image>` | Change encryption password |

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

Override base path: `ov config set encrypted_storage_path /path/to/storage` or `OV_ENCRYPTED_STORAGE_PATH=/path`.

## Commands

### Initialize

```bash
ov crypto init my-app                    # Init all encrypted volumes
ov crypto init my-app --volume secrets   # Init specific volume
```

Creates cipher directories and initializes gocryptfs. Prompts for password once (cached in kernel keyring for multi-volume images).

### Mount

```bash
ov crypto mount my-app                    # Mount all encrypted volumes
ov crypto mount my-app --volume secrets   # Mount specific volume
```

Prompts for password (or reuses from keyring). The plain directory becomes available for container bind mounts.

### Unmount

```bash
ov crypto unmount my-app                    # Unmount all
ov crypto unmount my-app --volume secrets   # Unmount specific
```

### Status

```bash
ov crypto status my-app
# secrets: mounted
# configs: not mounted
```

### Change Password

```bash
ov crypto passwd my-app
```

Changes the gocryptfs password for all encrypted volumes of an image.

## Single Password

When an image has multiple encrypted bind mounts, `ov crypto init`, `ov crypto mount`, and the generated crypto systemd unit all use `systemd-ask-password --id=ov-<image>` to cache the passphrase in the kernel keyring. Password is prompted once and reused for all volumes.

## Integration with Runtime

- **`ov shell`/`ov start` (direct mode)**: resolves bind mounts, verifies encrypted volumes are mounted, appends `-v <plain>:<container-path>` flags
- **`ov enable` (quadlet mode)**: generates a companion `ov-<image>-crypto.service` with `Requires=`/`After=` dependency. The crypto service mounts volumes before the main container starts
- **`ov remove`**: removes the companion crypto service file. `--volumes` also removes named volumes
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

Plain mounts do not use `ov crypto` commands. They are direct bind mounts.

Source: `ov/crypto.go`, `ov/validate.go` (`validateBindMounts`).

## Cross-References

- `/overthink:deploy` -- Quadlet integration, bind mount configuration, deploy.yml
- `/overthink:config` -- `encrypted_storage_path` setting
- `/overthink:service` -- Crypto companion service for quadlet mode

## When to Use This Skill

Use when the user asks about:

- `ov crypto` commands (init, mount, unmount, status, passwd)
- Encrypted bind mounts and gocryptfs
- Changing encryption passwords
- Encrypted storage paths
- "How do I encrypt container data?"
