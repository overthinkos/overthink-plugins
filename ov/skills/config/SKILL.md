---
name: config
description: |
  MUST be invoked before any work involving: ov config commands, runtime settings, engine selection, bind address, VM defaults, or encrypted storage paths.
---

# Config - Runtime Configuration

## Overview

`ov config` manages user-level runtime settings stored in `~/.config/ov/config.yml`. All settings follow a three-level resolution chain: **environment variable > config file > default**.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Get a value | `ov config get <key>` | Print resolved value |
| Set a value | `ov config set <key> <value>` | Write to config file |
| List all | `ov config list` | Show all keys with source |
| Reset a key | `ov config reset <key>` | Remove from config (revert to default) |
| Reset all | `ov config reset` | Clear entire config file |
| Show path | `ov config path` | Print config file path |
| Migrate secrets | `ov config migrate-secrets` | Move plaintext creds to system keyring |
| Dry-run migrate | `ov config migrate-secrets --dry-run` | Preview migration without changes |

## Config File

Location: `~/.config/ov/config.yml`

```yaml
engine:
  build: podman
  run: podman
  rootful: auto
run_mode: quadlet
auto_enable: true
bind_address: 127.0.0.1
encrypted_storage_path: ~/.local/share/ov/encrypted
secret_backend: auto
vm:
  backend: auto
  disk_size: "10 GiB"
  root_size: ""
  ram: "4G"
  cpus: 2
  rootfs: ext4
  transport: ""
```

## All Config Keys

### Engine Settings

| Key | Env Var | Default | Values | Purpose |
|-----|---------|---------|--------|---------|
| `engine.build` | `OV_BUILD_ENGINE` | `auto` | `auto`, `docker`, `podman` | Engine for `ov build` and `ov merge` |
| `engine.run` | `OV_RUN_ENGINE` | `auto` | `auto`, `docker`, `podman` | Engine for `ov shell`, `ov start`, `ov enable` |
| `engine.rootful` | `OV_ENGINE_ROOTFUL` | `auto` | `auto`, `machine`, `sudo`, `native` | Rootful podman strategy for VM builds |

Auto-detection prefers podman over docker when both are installed.

### Runtime Settings

| Key | Env Var | Default | Values | Purpose |
|-----|---------|---------|--------|---------|
| `run_mode` | `OV_RUN_MODE` | `direct` | `direct`, `quadlet` | Container lifecycle mode |
| `auto_enable` | `OV_AUTO_ENABLE` | `true` | `true`, `false` | Auto-generate quadlet on first `ov start` |
| `bind_address` | `OV_BIND_ADDRESS` | `127.0.0.1` | `127.0.0.1`, `0.0.0.0` | Address for port bindings |
| `encrypted_storage_path` | `OV_ENCRYPTED_STORAGE_PATH` | `~/.local/share/ov/encrypted` | any path | Gocryptfs storage base directory |
| `secret_backend` | `OV_SECRET_BACKEND` | `auto` | `auto`, `keyring`, `config` | Credential storage backend |

### VM Settings

| Key | Env Var | Default | Values | Purpose |
|-----|---------|---------|--------|---------|
| `vm.backend` | `OV_VM_BACKEND` | `auto` | `auto`, `libvirt`, `qemu` | VM hypervisor backend |
| `vm.disk_size` | `OV_VM_DISK_SIZE` | `10 GiB` | size string | Default disk image size |
| `vm.root_size` | `OV_VM_ROOT_SIZE` | (empty) | size string | Root partition size |
| `vm.ram` | `OV_VM_RAM` | `4G` | size string | Default VM memory |
| `vm.cpus` | `OV_VM_CPUS` | `2` | integer | Default VM CPU count |
| `vm.rootfs` | `OV_VM_ROOTFS` | `ext4` | `ext4`, `xfs`, `btrfs` | Root filesystem type |
| `vm.transport` | `OV_VM_TRANSPORT` | (empty) | `registry`, `containers-storage`, `oci`, `oci-archive` | Image transport for bootc builds |

### VNC Passwords

| Key | Env Var | Default | Values | Purpose |
|-----|---------|---------|--------|---------|
| `vnc.password.<image>` | `VNC_PASSWORD` | (empty) | any string | VNC auth password for image |
| `vnc.password.<image>-<instance>` | `VNC_PASSWORD` | (empty) | any string | VNC auth password for specific instance |

Set via `ov vnc passwd <image> --generate` or `ov config set vnc.password.<image> <password>`. Stored in system keyring (when `secret_backend=auto` or `keyring`) or `vnc_passwords` map in config file (when `secret_backend=config`).

### Sunshine Credentials

| Key | Env Var | Default | Values | Purpose |
|-----|---------|---------|--------|---------|
| `sunshine.user.<image>` | `SUNSHINE_USER` | (empty) | any string | Sunshine Web UI username |
| `sunshine.password.<image>` | `SUNSHINE_PASSWORD` | (empty) | any string | Sunshine Web UI password |
| `sunshine.user.<image>-<instance>` | `SUNSHINE_USER` | (empty) | any string | Instance-specific username |
| `sunshine.password.<image>-<instance>` | `SUNSHINE_PASSWORD` | (empty) | any string | Instance-specific password |

Set via `ov sun passwd <image> --generate` or `ov config set sunshine.user.<image> <user>` + `ov config set sunshine.password.<image> <pass>`. Stored in system keyring (when `secret_backend=auto` or `keyring`) or `sunshine_users`/`sunshine_passwords` maps in config file (when `secret_backend=config`).

## Resolution Chain

For every key, the resolved value is the first non-empty from:

1. **Environment variable** (e.g., `OV_BUILD_ENGINE`)
2. **Config file** (`~/.config/ov/config.yml`)
3. **Hardcoded default**

For credential keys (`vnc.password.*`, `sunshine.user.*`, `sunshine.password.*`), the chain is:

1. **Environment variable** (e.g., `VNC_PASSWORD`, `SUNSHINE_USER`, `SUNSHINE_PASSWORD`)
2. **System keyring** (when `secret_backend=auto` or `keyring`)
3. **Config file** (plaintext fallback)
4. **Default** (empty)

`ov config list` shows which source each value comes from. Credential values are masked with `****`.

## Common Workflows

### Switch to Podman

```bash
ov config set engine.build podman
ov config set engine.run podman
```

### Enable Quadlet Mode

```bash
ov config set run_mode quadlet
ov config set engine.run podman    # Required for quadlet
loginctl enable-linger $USER       # Required for user services
# auto_enable defaults to true -- ov start auto-generates quadlet on first run
```

### Override Via Environment

```bash
OV_BUILD_ENGINE=docker ov build my-image    # One-off override
export OV_RUN_MODE=quadlet                  # Session-wide override
```

### Check Current Settings

```bash
ov config list
# engine.build      auto     (default)
# engine.run        podman   (config)
# run_mode          direct   (default)
# ...
```

### Reset to Defaults

```bash
ov config reset engine.build    # Reset one key
ov config reset                 # Reset everything
```

### Migrate Credentials to System Keyring

```bash
ov config migrate-secrets --dry-run    # See what would be migrated
ov config migrate-secrets              # Actually migrate (creates .bak backup)
```

Migrates plaintext VNC and Sunshine credentials from `config.yml` to the system keyring (GNOME Keyring, KDE Wallet, KeePassXC). Creates a backup file first. If keyring is unavailable, prints setup instructions.

### Force Config-File Backend (headless/SSH)

```bash
ov config set secret_backend config    # Suppress keyring warnings
```

Source: `ov/runtime_config.go`, `ov/credential_store.go`, `ov/credential_keyring.go`, `ov/credential_config.go`.

## Cross-References

- `/ov:shell` -- Shell commands that use engine.run
- `/ov:service` -- Service lifecycle that uses run_mode
- `/ov:vm` -- VM commands that use vm.* settings
- `/ov:enc` -- Encrypted storage path configuration
- `/ov:build` -- Build commands that use engine.build
- `/ov:vnc` -- VNC password management (`vnc.password.*` keys)
- `/ov:sun` -- Sunshine credential management (`sunshine.user.*`, `sunshine.password.*` keys)

## When to Use This Skill

**MUST be invoked** when the task involves ov config commands, runtime settings, engine selection, bind address, VM defaults, encrypted storage paths, or credential storage backend. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Any time. Configuration can be set before or after builds/deployments.
