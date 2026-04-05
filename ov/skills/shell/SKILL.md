---
name: shell
description: |
  MUST be invoked before any work involving: ov shell command, interactive shells, command execution in containers, workspace mounts, TTY allocation, or port relay.
---

# Shell - Shell & Execution

## Overview

`ov shell` runs an interactive shell or executes commands inside containers. Supports workspace mounting, device auto-detection, environment injection, and TTY allocation for automation tools.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Interactive shell | `ov shell <image>` | Bash shell in container |
| Bind workspace | `ov shell <image> --bind workspace=~/project` | Mount host dir as workspace volume |
| Run command | `ov shell <image> -c "cmd"` | Execute command and exit |
| Force TTY | `ov shell <image> --tty -c "cmd"` | PTY allocation for automation |
| Specific version | `ov shell <image> --tag v1.0.0` | Use specific image tag |
| No devices | `ov shell <image> --no-autodetect` | Disable device auto-detection |
| Set env var | `ov shell <image> -e KEY=VALUE` | Inject environment variable |
| Load env file | `ov shell <image> --env-file .env` | Load env from file |
| Named instance | `ov shell <image> -i runner-1` | Use named instance |
| Force build | `ov shell <image> --build` | Build locally before running |
| Remote ref | `ov shell github.com/org/repo/app` | Pull and run remote image |

## How It Works

1. Resolves image reference (local images.yml or remote `github.com/...`)
2. Ensures image exists in run engine (transfers from build engine if needed)
3. Resolves volumes (with deploy-time backing), ports, security, environment
4. If container is already running: `<engine> exec` into it
5. If not running: `<engine> run` with full configuration

## Volume Binding

```bash
ov shell <image> --bind workspace=~/project  # Bind ~/project as workspace volume
ov shell <image> --bind workspace=.          # Bind PWD as workspace volume
ov shell <image>                             # No host mount (named volume)
```

The `--bind` flag overrides volume backing for the session. The container working directory is `~/workspace` (resolved from the `workspace` volume). If a `.env` file exists in the bound directory, it is auto-loaded.

## --tty Flag

Forces TTY allocation even when stdout is not a terminal. Uses `script(1)` from util-linux to provide a real PTY.

```bash
# Without --tty: "not a tty" errors from interactive commands
ov shell my-app -c "interactive-cli auth login"       # fails in CI/automation

# With --tty: script(1) wraps the command in a PTY
ov shell my-app --tty -c "interactive-cli auth login"  # works everywhere
```

Essential for automation tools (Claude Code, CI runners) where stdout is a pipe, not a terminal. Many interactive CLI commands (OAuth flows, prompts) require a TTY.

Works for both new containers (`<engine> run`) and exec into running containers (`<engine> exec`).

## Exec Into Running Containers

When `ov shell` detects the container is already running (via `<engine> inspect`), it uses `<engine> exec` instead of `<engine> run`. The same flags work:

```bash
ov start my-app                            # Start background service
ov shell my-app -c "cat /etc/hostname"     # Exec into running container
ov shell my-app                            # Interactive shell in running container
```

## Port Relay

The `port_relay` field in layer.yml solves the problem of services that bind only to 127.0.0.1 inside the container (e.g., Chrome 146+ DevTools). Container port mappings only forward to interfaces visible from outside, so a loopback-only service is unreachable from the host.

```yaml
# In layer.yml:
port_relay:
  - 9222
```

How it works:
- socat binds to `eth0:<port>` and forwards to `127.0.0.1:<port>`
- An init system service is auto-generated (via init.yml `relay_template`) with priority 1 (starts before other services)
- The socat layer is automatically added as a dependency

This makes loopback-only services accessible through normal podman/docker port mappings and `tailscale serve`.

Source: `ov/generate.go` (relay service generation), `ov/layers.go` (`PortRelayYAML`).

## Device Auto-Detection

By default, `ov shell` auto-detects available host devices and passes them through. Use `--no-autodetect` to disable.

**Auto-detected devices:**
- NVIDIA GPU (via `nvidia-smi`) -- `--gpus all` (Docker) or `--device nvidia.com/gpu=all` (Podman)
- AMD GPU (via sysfs `amdgpu` driver) -- `--device /dev/kfd` + `--group-add keep-groups` + auto-detected `HSA_OVERRIDE_GFX_VERSION` from KFD topology
- `/dev/dri/renderD*` -- GPU render nodes
- `/dev/kfd` -- AMD Kernel Fusion Driver (ROCm compute)
- `/dev/kvm` -- KVM virtualization
- `/dev/vhost-net`, `/dev/vhost-vsock` -- virtio networking
- `/dev/fuse` -- FUSE filesystem
- `/dev/net/tun` -- TUN/TAP networking
- `/dev/hwrng` -- hardware RNG

When an AMD GPU is detected, `keep-groups` is auto-added to preserve host supplementary groups (video, render) inside the container, and `HSA_OVERRIDE_GFX_VERSION` is auto-set from the GPU's KFD topology (e.g., `10.3.0` for RDNA2). The HSA env var can be overridden via `-e`.

Source: `ov/devices.go` (`DetectHostDevices`, `DetectGPU`, `DetectAMDGPU`).

## Environment Variables

Runtime environment variables are injected from multiple sources. Resolution priority (last wins for duplicate keys):

1. **Deploy config `env:`** (images.yml / deploy.yml) -- lowest priority
2. **Deploy config `env_file:`** (images.yml / deploy.yml)
3. **Workspace `.env`** file -- auto-loaded from `-w` directory
4. **CLI `--env-file`** flag
5. **CLI `-e`** flags -- highest priority

```bash
ov shell <image> -e DB_HOST=localhost -e DB_PORT=5432
ov shell <image> --env-file production.env
```

`.env` file format (Docker-compatible): `KEY=VALUE`, `KEY="VALUE"`, `KEY='VALUE'`, `KEY` (inherits from host), `#` comments, blank lines ignored.

Source: `ov/envfile.go` (`ParseEnvFile`, `ResolveEnvVars`, `LoadWorkspaceEnv`).

## Remote Image References

`ov shell` accepts remote image references: `github.com/org/repo/image[@version]`.

```bash
ov shell github.com/org/repo/my-app               # Pull and run
ov shell github.com/org/repo/my-app@v1.0.0        # Specific version
ov shell github.com/org/repo/my-app --build        # Force local build
```

**Registry-first approach:** attempts to pull the pre-built image from the remote project's registry. Falls back to local build (download repo, generate, build). `--build` skips pull and always builds locally.

Source: `ov/remote_image.go`.

## Cross-Engine Image Transfer

When `engine.build` differs from `engine.run`, images are automatically transferred between engines on demand via `<src> save | <dst> load`.

Source: `ov/transfer.go`.

## Container Networking

All containers are connected to a shared `ov` network by default, enabling inter-container DNS resolution by container name. Override with `network: host` in images.yml.

Source: `ov/network.go`.

## `ov cmd` vs `ov shell -c`

`ov cmd <image> "command"` is a dedicated single-command tool for **running containers only**. Key differences from `ov shell -c`:

| | `ov cmd` | `ov shell -c` |
|---|---------|--------------|
| Container state | Running only | Running or starts new |
| Notification | Yes (`--[no-]notify`) | No |
| Process model | `exec.Command` (returns) | `syscall.Exec` (replaces) |
| Workspace mount | No | Yes (`-w`) |
| Device auto-detect | No | Yes |
| Use case | Quick commands + notification | Full container setup + command |

Use `ov cmd` for quick operations on running services. Use `ov shell -c` when you need workspace mounts, device passthrough, or need to start a new container.

## Cross-References

- `ov cmd` -- Single command execution with D-Bus notification (running containers only)
- `/ov:tmux` -- Persistent tmux sessions (survives disconnects, needed for TTY-dependent TUI programs)
- `/ov:service` -- Starting background services before exec
- `/ov:cdp` -- Chrome DevTools Protocol automation
- `/ov:wl` (sway subgroup) -- Sway compositor control
- `/ov:config` -- Engine and bind address settings
- `/ov:image` -- Image configuration (ports, volumes, env)

## When to Use This Skill

**MUST be invoked** when the task involves ov shell command, interactive shells, command execution in containers, workspace mounts, TTY allocation, or port relay. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Any time. Interactive access to running or stopped containers.
