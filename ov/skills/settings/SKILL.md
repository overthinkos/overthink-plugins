---
name: settings
description: |
  Runtime configuration management for the ov CLI.
  MUST be invoked before any work involving: ov settings commands, runtime configuration, engine selection, bind address, storage paths, or secret backend configuration.
---

# ov settings -- Runtime Configuration

## Overview

Manage ov's runtime configuration stored in `~/.config/ov/settings.yml`. Controls engine selection, networking, storage paths, secret backend, and agent forwarding.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Get a setting | `ov settings get <key>` | Show current value |
| Set a setting | `ov settings set <key> <value>` | Update a setting |
| List all | `ov settings list` | Show all settings with values |
| Reset to default | `ov settings reset <key>` | Remove override, use default |
| Config path | `ov settings path` | Print path to settings.yml |
| Migrate secrets | `ov settings migrate-secrets [--dry-run]` | Move plaintext credentials to keyring |

## Key Settings

| Key | Default | Env Var | Description |
|-----|---------|---------|-------------|
| `engine.build` | `docker` | `OV_ENGINE_BUILD` | Build engine (docker/podman) |
| `engine.run` | `docker` | `OV_ENGINE_RUN` | Run engine (docker/podman) |
| `run_mode` | `quadlet` | `OV_RUN_MODE` | Deployment mode (quadlet/direct) |
| `bind_address` | `127.0.0.1` | `OV_BIND_ADDRESS` | Default bind address for ports |
| `encrypted_storage_path` | `~/.local/share/ov/encrypted` | `OV_ENCRYPTED_STORAGE_PATH` | Base path for gocryptfs volumes |
| `volumes_path` | `~/.local/share/ov/volumes` | `OV_VOLUMES_PATH` | Base path for bind-mounted volumes |
| `secret_backend` | `auto` | `OV_SECRET_BACKEND` | Credential backend (auto/keyring/kdbx/config) |
| `forward_gpg_agent` | `true` | `OV_FORWARD_GPG_AGENT` | Forward GPG agent into containers |
| `forward_ssh_agent` | `true` | `OV_FORWARD_SSH_AGENT` | Forward SSH agent into containers |
| `secrets.kdbx_path` | *(none)* | `OV_KDBX_PATH` | Path to KeePass .kdbx database |
| `secrets.kdbx_cache` | `true` | `OV_KDBX_CACHE` | Cache kdbx password in kernel keyring |
| `secrets.kdbx_cache_timeout` | `3600` | `OV_KDBX_CACHE_TIMEOUT` | Kernel keyring cache TTL (seconds) |

## Usage

### Engine Selection

```bash
# Switch to podman for both build and run
ov settings set engine.build podman
ov settings set engine.run podman

# Check current engine
ov settings get engine.build
```

### Storage Paths

```bash
# Change volume storage to NAS
ov settings set volumes_path /mnt/nas/ov-volumes

# Change encrypted storage location
ov settings set encrypted_storage_path /mnt/encrypted/ov
```

### Secret Backend

```bash
# Force KeePass backend
ov settings set secret_backend kdbx

# Migrate plaintext secrets from settings.yml to keyring
ov settings migrate-secrets

# Preview migration without changes
ov settings migrate-secrets --dry-run
```

### Resolution Chain

Settings resolve in this order: environment variable > settings.yml > default value.

## Cross-References

- `/ov:config` -- deployment configuration (uses settings)
- `/ov:secrets` -- credential management
- `/ov:doctor` -- diagnose settings and secret storage health
- `/ov:enc` -- encrypted volume paths
