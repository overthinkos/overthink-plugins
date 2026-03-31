---
name: enc
description: |
  MUST be invoked before any work involving: encrypted storage, ov config mount/unmount/status/passwd commands, gocryptfs, or encrypted volume backing.
---

# Enc - Encrypted Storage

## Overview

Encrypted volume backing is configured at deploy time via `ov config --encrypt <volume>`. Gocryptfs-encrypted volumes store sensitive data (credentials, keys, configs) with transparent encryption at rest. The cipher directory lives on disk; the plain directory is mounted on demand. `ov config <image>` handles initialization and mounting during deployment setup. `ov start` mounts encrypted volumes inline before starting the container.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Configure encrypted | `ov config <image> --encrypt <vol>` | Set volume backing to encrypted |
| Setup (init + mount) | `ov config <image>` | Initialize cipher dirs and mount encrypted volumes |
| Mount | `ov config mount <image>` | Mount encrypted volumes |
| Unmount | `ov config unmount <image>` | Unmount encrypted volumes |
| Status | `ov config status <image>` | Show mount status |
| Change password | `ov config passwd <image>` | Change encryption password |

All commands accept `--volume NAME` to target a specific volume (otherwise all encrypted volumes are affected).

## Configuration

Encrypted volumes are configured at deploy time, not build time. Use `ov config --encrypt` to set a layer-declared volume's backing to encrypted:

```bash
# Configure "library" volume as encrypted
ov config immich --encrypt library

# Or via canonical syntax
ov config immich -v library:encrypted

# Or via env var
OV_VOLUMES_IMMICH="library:encrypted" ov config immich --password auto
```

This saves to deploy.yml:
```yaml
volumes:
  - name: library
    type: encrypted
```

Rules:
- Volume must be declared in `layer.yml` (or provided as deploy-only with `path:`)
- `name`: matches a layer volume name, must match `^[a-z0-9]+(-[a-z0-9]+)*$`

### Per-Volume Explicit Path

Each encrypted volume can specify its own storage directory:

```bash
# Explicit per-volume paths (each volume gets its own directory)
ov config immich-ml \
  --volume library:encrypt:/mnt/nas/immich/library \
  --volume pgdata:encrypt:/mnt/nas/immich/pgdata

# Canonical syntax
ov config immich-ml -v library:encrypted:/mnt/nas/immich/library
```

The path is the **direct volume directory** — `cipher/` and `plain/` are created inside it:
```
/mnt/nas/immich/library/
  cipher/    # gocryptfs encrypted data
  plain/     # FUSE mount point
```

Without an explicit path, the global `encrypted_storage_path` is used with an `ov-<image>-<name>` prefix (backward compatible).

## Storage Layout

### Default (no explicit path)
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
ov config my-app --encrypt secrets              # Set volume as encrypted + init + mount
ov config my-app --password auto                # Auto-generate password
ov config my-app --password manual              # Prompt for password
```

`ov config <image>` handles both initialization (creating cipher directories) and mounting in a single step. If volumes are already initialized, it mounts them. Password is cached in kernel keyring for multi-volume images.

### Mount

```bash
ov config mount my-app                    # Mount all encrypted volumes
ov config mount my-app --volume secrets   # Mount specific volume
```

Prompts for password (or reuses from keyring). The plain directory becomes available for container use.

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

When an image has multiple encrypted volumes, `ov config`, `ov config mount`, and `ov start` all use `systemd-ask-password --id=ov-<image>` to cache the passphrase in the kernel keyring. Password is prompted once and reused for all volumes.

## KeePass Integration

Use the `--kdbx` global flag to specify a KeePass database for password storage:

```bash
ov --kdbx ~/.config/ov/secrets.kdbx config my-app --encrypt secrets
```

## Integration with Runtime

- **`ov shell`/`ov start` (direct mode)**: resolves volume backing from deploy.yml, verifies encrypted volumes are mounted, appends `-v <plain>:<container-path>` flags. `ov start` mounts encrypted volumes inline before starting the container
- **`ov config` (quadlet mode)**: generates quadlet file with `ExecStartPre=ov config mount <image>` for encrypted services. Boot behavior is backend-gated (see below)
- **`ov remove --purge`**: removes named volumes
- **`ov seed <image>`**: copies default data from the image into empty bind-backed directories (works for both bind and encrypted volumes after mounting)

### Boot Behavior: Backend-Gated

| Credential Backend | Quadlet Behavior | User Action on Reboot |
|---|---|---|
| **Secret Service (keyring)** | `WantedBy=default.target` + `ExecStartPre` (waits for keyring) + `TimeoutStartSec=0` | None — auto-starts after login |
| **KeePass (.kdbx)** | `ExecStartPre` (guard) + NO `WantedBy` | `ov start <image>` (prompts for kdbx master) |
| **Config file / none** | `ExecStartPre` (guard) + NO `WantedBy` | `ov start <image>` (prompts interactively) |

**Secret Service flow on reboot:**
1. Boot → systemd starts user instance (linger) → quadlet service starts
2. ExecStartPre → `ov config mount` → keyring locked → polls every 5s
3. User logs in → PAM unlocks GNOME Keyring
4. Next poll → passphrase found → volumes mount → container starts

**Crash recovery:** FUSE mounts survive container restarts → ExecStartPre detects already-mounted → skips → container restarts immediately

## Volume Backing Override

When a volume is configured as `type: encrypted` in deploy.yml, it overrides the default named volume. The Docker/Podman named volume is not created -- the gocryptfs mount is used instead.

```yaml
# layer.yml declares a volume:
volumes:
  - name: data
    path: "~/.myapp"

# deploy.yml configures it as encrypted:
volumes:
  - name: data
    type: encrypted
```

## Plain (Non-Encrypted) Bind Mounts

For comparison, plain bind mounts use `type: bind`:

```bash
ov config my-app --bind data=/mnt/nas/data    # Explicit host path
ov config my-app --bind data                   # Auto path: ~/.local/share/ov/volumes/my-app/data
```

Plain bind mounts do not use encrypted storage commands. They are direct host directory mounts.

Source: `ov/enc.go` (encrypted lifecycle), `ov/deploy.go` (`DeployVolumeConfig`, `ResolveVolumeBacking`).

## Cross-References

- `/ov:deploy` -- Quadlet integration, volume backing configuration, deploy.yml
- `/ov:config` -- `encrypted_storage_path` and `volumes_path` settings
- `/ov:service` -- Container lifecycle, `ov start` inline mount

## When to Use This Skill

**MUST be invoked** when the task involves encrypted storage, gocryptfs, or encrypted volume backing. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Pre-deployment. Configure encrypted volumes during `ov config`. See also `/ov:deploy` (volume backing).
