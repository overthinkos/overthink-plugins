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

## Runtime Configuration

```bash
ov config set engine.build docker    # Build engine
ov config set engine.run docker      # Run engine
ov config set run_mode quadlet       # Service mode
ov config set bind_address 0.0.0.0   # Port binding address
ov config list                       # Show all settings
```

## Cross-Engine Image Transfer

When `engine.build` differs from `engine.run`, images are automatically transferred via `docker save | podman load` (or reverse).

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
