---
name: config
description: |
  MUST be invoked before any work involving: ov settings commands, runtime settings, engine selection, bind address, VM defaults, or encrypted storage paths.
---

# Config - Runtime Configuration

## Overview

`ov settings` manages user-level runtime settings stored in `~/.config/ov/config.yml`. All settings follow a three-level resolution chain: **environment variable > config file > default**.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Get a value | `ov settings get <key>` | Print resolved value |
| Set a value | `ov settings set <key> <value>` | Write to config file |
| List all | `ov settings list` | Show all keys with source |
| Reset a key | `ov settings reset <key>` | Remove from config (revert to default) |
| Reset all | `ov settings reset` | Clear entire config file |
| Show path | `ov settings path` | Print config file path |
| Migrate secrets | `ov settings migrate-secrets` | Move plaintext creds to system keyring |
| Dry-run migrate | `ov settings migrate-secrets --dry-run` | Preview migration without changes |

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
| `engine.run` | `OV_RUN_ENGINE` | `auto` | `auto`, `docker`, `podman` | Engine for `ov shell`, `ov start`, `ov config` |
| `engine.rootful` | `OV_ENGINE_ROOTFUL` | `auto` | `auto`, `machine`, `sudo`, `native` | Rootful podman strategy for VM builds |

Auto-detection prefers podman over docker when both are installed.

### Runtime Settings

| Key | Env Var | Default | Values | Purpose |
|-----|---------|---------|--------|---------|
| `run_mode` | `OV_RUN_MODE` | `direct` | `direct`, `quadlet` | Container lifecycle mode |
| `auto_enable` | `OV_AUTO_ENABLE` | `true` | `true`, `false` | Auto-generate quadlet on first `ov start` |
| `bind_address` | `OV_BIND_ADDRESS` | `127.0.0.1` | `127.0.0.1`, `0.0.0.0` | Address for port bindings |
| `encrypted_storage_path` | `OV_ENCRYPTED_STORAGE_PATH` | `~/.local/share/ov/encrypted` | any path | Gocryptfs storage base directory |
| `secret_backend` | `OV_SECRET_BACKEND` | `auto` | `auto`, `keyring`, `kdbx`, `config` | Credential storage backend |
| `secrets.kdbx_path` | `OV_KDBX_PATH` | (empty) | any path | Path to KeePass .kdbx database |
| `secrets.kdbx_key_file` | `OV_KDBX_KEY_FILE` | (empty) | any path | Optional key file for .kdbx |

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

Set via `ov vnc passwd <image> --generate` or `ov settings set vnc.password.<image> <password>`. Stored in system keyring (when `secret_backend=auto` or `keyring`) or `vnc_passwords` map in config file (when `secret_backend=config`).

## Resolution Chain

For every key, the resolved value is the first non-empty from:

1. **Environment variable** (e.g., `OV_BUILD_ENGINE`)
2. **Config file** (`~/.config/ov/config.yml`)
3. **Hardcoded default**

For credential keys (`vnc.password.*`), the chain is:

1. **Environment variable** (e.g., `VNC_PASSWORD`)
2. **System keyring** (when `secret_backend=auto` or `keyring`)
3. **KeePass .kdbx** (when `secret_backend=kdbx`, or auto-detected when keyring unavailable and `secrets.kdbx_path` configured)
4. **Config file** (plaintext fallback)
5. **Default** (empty)

`ov settings list` shows which source each value comes from. Credential values are masked with `****`.

## Common Workflows

### Switch to Podman

```bash
ov settings set engine.build podman
ov settings set engine.run podman
```

### Enable Quadlet Mode

```bash
ov settings set run_mode quadlet
ov settings set engine.run podman    # Required for quadlet
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
ov settings list
# engine.build      auto     (default)
# engine.run        podman   (config)
# run_mode          direct   (default)
# ...
```

### Reset to Defaults

```bash
ov settings reset engine.build    # Reset one key
ov settings reset                 # Reset everything
```

### Migrate Credentials to System Keyring

```bash
ov settings migrate-secrets --dry-run    # See what would be migrated
ov settings migrate-secrets              # Actually migrate (creates .bak backup)
```

Migrates plaintext VNC credentials from `config.yml` to the system keyring (GNOME Keyring, KDE Wallet, KeePassXC). Creates a backup file first. If keyring is unavailable, prints setup instructions.

### KeePass Backend (headless/SSH -- encrypted, no daemon)

```bash
ov secrets init                              # Create ~/.config/ov/secrets.kdbx
ov settings set secret_backend kdbx          # Activate (or auto-detected when keyring unavailable)
ov secrets import --dry-run                  # Preview importing existing credentials
ov secrets import                            # Import from config.yml + keyring into kdbx
```

Encrypted at rest (KDBX 4, Argon2). No daemon needed. Works over SSH. Password prompted once per session (cached in kernel keyring via `systemd-ask-password`).

### Force Config-File Backend (headless/SSH)

```bash
ov settings set secret_backend config    # Suppress keyring warnings
```

### Global KeePass Flag

```bash
ov --kdbx ~/.config/ov/secrets.kdbx start my-app    # Use kdbx for this command
```

Source: `ov/runtime_config.go`, `ov/credential_store.go`, `ov/credential_keyring.go`, `ov/credential_config.go`, `ov/credential_kdbx.go`, `ov/secrets_cmd.go`.

## Cross-References

- `/ov:shell` -- Shell commands that use engine.run
- `/ov:service` -- Service lifecycle that uses run_mode
- `/ov:vm` -- VM commands that use vm.* settings
- `/ov:enc` -- Encrypted storage path configuration
- `/ov:build` -- Build commands that use engine.build
- `/ov:vnc` -- VNC password management (`vnc.password.*` keys)

## When to Use This Skill

**MUST be invoked** when the task involves ov settings commands, runtime settings, engine selection, bind address, VM defaults, encrypted storage paths, or credential storage backend. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Any time. Configuration can be set before or after builds/deployments.
