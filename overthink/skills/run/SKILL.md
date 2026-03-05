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
| Seed bind mounts | `ov seed <image>` | Copy image data to empty bind mount dirs |
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
| `task run:vm -- <image>` | Create VM via `ov vm create` |
| `task run:vm-start -- <image>` | Start VM |
| `task run:vm-stop -- <image>` | Stop VM |
| `task run:vm-destroy -- <image>` | Destroy VM |
| `task run:vm-list` | List VMs |
| `task run:vm-console -- <image>` | VM serial console |
| `task run:vm-ssh -- <image>` | SSH into VM |

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
ov shell <image> -e MY_VAR=value          # Set env var
ov shell <image> --env-file .env          # Load env file
ov shell <image> -i runner-1              # Named instance
ov shell <image> --build                  # Force local build
ov shell github.com/org/repo/app          # Remote image ref
```

Features:
- Mounts current directory at `/workspace` by default
- Automatically mounts declared volumes (`ov-<image>-<name>`)
- Resolves bind mounts (plain and encrypted)
- Applies port mappings from image config
- Transfers image between engines if needed
- Applies security config (privileged, capabilities, devices)

## Service Lifecycle

```bash
ov start <image>                  # Start background service
ov start <image> -i prod          # Start named instance
ov status <image>                 # Check if running
ov logs <image> -f                # Follow logs
ov stop <image>                   # Stop service
ov update <image>                 # Rebuild + restart
ov update <image> --build         # Force local rebuild + restart
ov remove <image>                 # Stop + cleanup
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

## Environment Variables

Runtime environment variables can be injected into containers from multiple sources. Resolution priority (last wins for duplicate keys):

1. **Deploy config `env:`** (images.yml / deploy.yml) -- lowest priority
2. **Deploy config `env_file:`** (images.yml / deploy.yml)
3. **Workspace `.env`** file -- auto-loaded from `-w` directory
4. **CLI `--env-file`** flag
5. **CLI `-e`** flags -- highest priority

```bash
ov shell <image> -e DB_HOST=localhost -e DB_PORT=5432
ov shell <image> --env-file production.env
ov start <image> -e LOG_LEVEL=debug --env-file base.env
```

`.env` file format (Docker-compatible): `KEY=VALUE`, `KEY="VALUE"`, `KEY='VALUE'`, `KEY` (inherits from host), `#` comments, blank lines ignored.

Source: `ov/envfile.go` (`ParseEnvFile`, `ResolveEnvVars`, `LoadWorkspaceEnv`).

## Multi-Instance Support

The `--instance NAME` (`-i`) flag enables running multiple containers of the same image with separate state:

```bash
ov enable <image> -i prod -w ~/prod          # Instance "prod"
ov enable <image> -i staging -w ~/staging    # Instance "staging"
ov start <image> -i prod
ov status <image> -i staging
```

Instance naming affects:
- Container name: `ov-<image>` → `ov-<image>-<instance>`
- Volume names: `ov-<image>-<name>` → `ov-<image>-<instance>-<name>`
- Quadlet file: `ov-<image>.container` → `ov-<image>-<instance>.container`
- Service name: `ov-<image>.service` → `ov-<image>-<instance>.service`

Available on: `shell`, `start`, `stop`, `enable`, `disable`, `status`, `logs`, `update`, `remove`.

Source: `ov/volumes.go` (`InstanceVolumes`), `ov/quadlet.go`.

## Remote Image References

All runtime commands accept remote image refs: `github.com/org/repo/image[@version]`.

```bash
ov shell github.com/org/repo/my-app              # Pull and run
ov shell github.com/org/repo/my-app@v1.0.0       # Specific version
ov enable github.com/org/repo/my-app --build      # Force local build
```

**Registry-first approach:** attempts to pull the pre-built image from the remote project's registry. Falls back to local build (download repo, generate Containerfiles, build). `--build` flag skips the pull attempt and always builds locally.

Source: `ov/remote_image.go` (`ResolveRemoteImage`, `PullOrBuild`).

## Seed Command

`ov seed <image>` copies image data into empty bind mount directories that override layer volumes.

```bash
ov seed my-app                 # Seed all empty bind mount dirs
ov seed my-app --tag v1.0.0    # From specific image version
```

Only seeds bind mounts that match layer volume names (the override mechanism). Skips directories that already contain data. Useful for initializing bind mount directories on first deployment.

Source: `ov/seed.go`.

## VM Management

For bootc images, `ov vm` commands manage virtual machines built from disk images:

```bash
ov vm create my-image                  # Create VM from disk image
ov vm create my-image --ram 8G --cpus 4  # Custom resources
ov vm create my-image -i prod          # Named instance
ov vm start my-image                   # Start a stopped VM
ov vm stop my-image                    # Graceful shutdown
ov vm destroy my-image --disk          # Remove VM and delete disk
ov vm list -a                          # List all VMs
ov vm console my-image                 # Serial console
ov vm ssh my-image -p 2222 -l user     # SSH access
```

See `/overthink:deploy` for full VM configuration, backends, and libvirt XML injection.

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
- Environment variable injection (`-e`, `--env-file`)
- Multi-instance containers (`--instance`)
- Remote image references
- Seeding bind mount data (`ov seed`)
- "How do I run X in a container?"
