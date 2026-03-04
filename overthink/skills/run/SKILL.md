---
name: run
description: |
  Runtime operations: running shells, starting/stopping services, managing
  containers. Use when working with ov shell, ov start/stop, or task run:*
  commands.
---

# Run - Runtime Operations

## Overview

The `ov` CLI provides runtime commands for interacting with built container images: running interactive shells, starting background services, managing service lifecycle, and executing commands inside containers. Supports both direct mode (Docker/Podman run) and quadlet mode (systemd user services).

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Shell into image | `ov shell <image>` | Bash shell in a container |
| Shell with workspace | `ov shell <image> -w ~/project` | Mount directory at /workspace |
| Run command | `ov shell <image> -c "cmd"` | Execute command and exit |
| Start service | `ov start <image>` | Start as background service |
| Stop service | `ov stop <image>` | Stop running service |
| Service status | `ov status <image>` | Show service status |
| Service logs | `ov logs <image> -f` | Follow service logs |
| Update service | `ov update <image>` | Update image and restart |
| Remove service | `ov remove <image>` | Stop and remove service |
| Install aliases | `ov alias install <image>` | Create host command wrappers |
| Uninstall aliases | `ov alias uninstall <image>` | Remove host command wrappers |

## Task Wrappers

| Task | Description |
|------|-------------|
| `task run:shell -- <image>` | Shell into image |
| `task run:start -- <image>` | Start service |
| `task run:stop -- <image>` | Stop service |
| `task run:status -- <image>` | Service status |
| `task run:logs -- <image>` | Service logs |
| `task run:update -- <image>` | Update and restart |
| `task run:remove -- <image>` | Remove service |
| `task run:enable -- <image>` | Enable quadlet service |
| `task run:disable -- <image>` | Disable quadlet service |
| `task run:vm -- <image>` | Run QCOW2 in QEMU |

## Run Modes

| Mode | Config | How it works |
|------|--------|-------------|
| `direct` | `run_mode: direct` | `<engine> run -d` / `<engine> stop` |
| `quadlet` | `run_mode: quadlet` | systemd user services via podman quadlet |

```bash
ov config set run_mode direct   # Default
ov config set run_mode quadlet  # systemd integration
```

## GPU Passthrough

| Flag | Behavior |
|------|----------|
| `--gpu` | Force GPU passthrough |
| `--no-gpu` | Disable GPU passthrough |
| (neither) | Auto-detect via `nvidia-smi` |

Engine-specific flags:
- **Docker:** `--gpus all`
- **Podman:** `--device nvidia.com/gpu=all`

## Shell Command

```bash
ov shell <image>                          # Interactive bash
ov shell <image> -w ~/project             # Mount workspace
ov shell <image> -c "nvidia-smi"          # Run command
ov shell <image> --tag 2026.46.1415       # Specific version
ov shell <image> --gpu                    # Force GPU
```

Features:
- Mounts current directory at `/workspace` by default
- Automatically mounts declared volumes (`ov-<image>-<name>`)
- Resolves bind mounts (plain and encrypted)
- Applies port mappings from image config
- Transfers image between engines if needed

## Service Lifecycle

```bash
ov start <image>        # Start background service
ov status <image>       # Check if running
ov logs <image> -f      # Follow logs
ov stop <image>         # Stop service
ov update <image>       # Rebuild + restart
ov remove <image>       # Stop + cleanup
```

## Command Aliases

Distrobox-style wrapper scripts for transparent host-to-container commands:

```bash
ov alias install openclaw     # Creates ~/.local/bin/openclaw
openclaw                      # Runs inside container transparently
ov alias uninstall openclaw   # Removes wrapper
ov alias list                 # Show all installed aliases
```

### Wrapper Script Format

`ov alias add`/`ov alias install` writes shell scripts to `~/.local/bin/`:

```sh
#!/bin/sh
# ov-alias
# image: openclaw
# command: openclaw
_ov_q(){ printf "'"; printf '%s' "$1" | sed "s/'/'\\\\''/g"; printf "' "; }
c="openclaw"; for a in "$@"; do c="$c $(_ov_q "$a")"; done
exec ov shell openclaw -c "$c"
```

The `# ov-alias` marker enables safe list/delete scanning. `ov alias remove` verifies this marker before deleting. The `_ov_q()` helper properly single-quotes each argument (POSIX sh compatible).

### Declaration

Layer aliases require both `name` and `command`. Image-level aliases default `command` to `name` if omitted. Image-level aliases override layer aliases with the same name.

`CollectImageAliases()` gathers aliases from the image's own layers (in dependency order) plus image-level config. **No base chain traversal** -- aliases are leaf-image specific (unlike volumes).

Source: `ov/alias.go`, `ov/layers.go` (`AliasYAML`, `HasAliases`, `Aliases()`).

## Runtime Configuration

```bash
ov config set engine.build docker    # Build engine
ov config set engine.run docker      # Run engine
ov config set run_mode quadlet       # Service mode
ov config set bind_address 0.0.0.0   # Port binding address
ov config list                       # Show all settings
```

Stored in `~/.config/ov/config.yml`. Resolution chain: **env var > config file > default**.

| Setting | Env Var | Default | Purpose |
|---------|---------|---------|---------|
| `engine.build` | `OV_BUILD_ENGINE` | `docker` | Engine for `ov build` and `ov merge` |
| `engine.run` | `OV_RUN_ENGINE` | `docker` | Engine for `ov shell` and `ov start` |
| `run_mode` | `OV_RUN_MODE` | `direct` | `direct` or `quadlet` |
| `auto_enable` | `OV_AUTO_ENABLE` | `false` | Auto-run `ov enable` on first `ov start` (quadlet) |
| `bind_address` | `OV_BIND_ADDRESS` | `127.0.0.1` | Address for port bindings |
| `encrypted_storage_path` | `OV_ENCRYPTED_STORAGE_PATH` | `~/.local/share/ov/encrypted` | Gocryptfs storage base |

When `run_mode=quadlet`, `ov start` checks for an existing `.container` file. If none exists and `auto_enable=true`, it auto-enables. If `auto_enable=false`, it errors with a message to run `ov enable` first. Quadlet requires `engine.run=podman`.

Source: `ov/runtime_config.go`, `ov/engine.go`.

## Cross-Engine Image Transfer

When `engine.build` differs from `engine.run`, images are automatically transferred between engines on demand.

| Function | Purpose |
|----------|---------|
| `LocalImageExists(engine, imageRef)` | Check if image exists in engine's local store |
| `TransferImage(srcEngine, dstEngine, imageRef)` | Bidirectional pipe: `<src> save <ref> \| <dst> load` |
| `EnsureImage(imageRef, rt)` | Transfer from build to run engine if needed, error if missing from both |

Transfer is called automatically by `ov shell`, `ov start`, `ov enable`, and `ov update`.

Source: `ov/transfer.go`.

## Cross-References

- `/overthink:build` -- Building images before running
- `/overthink:deploy` -- Quadlet and deployment details
- `/overthink:image` -- Image configuration (ports, volumes, aliases)

## When to Use This Skill

Use when the user asks about:

- Running containers or shells (`ov shell`, `ov start`)
- Service management (start, stop, status, logs)
- GPU passthrough configuration
- Command aliases
- Runtime configuration
- "How do I run X in a container?"
