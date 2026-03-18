---
name: run
description: |
  Runtime operations: running shells, starting/stopping services, managing
  containers. Use when working with ov shell, ov start/stop commands.
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
| Remove with volumes | `ov remove <image> --volumes` | Stop, remove, and delete named volumes |
| Seed bind mounts | `ov seed <image>` | Copy image data to empty bind mount dirs |
| Install aliases | `ov alias install <image>` | Create host command wrappers |
| Uninstall aliases | `ov alias uninstall <image>` | Remove host command wrappers |
| Service status | `ov service status <image>` | Show supervisord service status |
| Start service | `ov service start <image> <svc>` | Start a supervisord service |
| Stop service | `ov service stop <image> <svc>` | Stop a supervisord service |
| Restart service | `ov service restart <image> <svc>` | Restart a supervisord service |

## Run Modes

| Mode | Config | How it works |
|------|--------|-------------|
| `direct` | `run_mode: direct` | `<engine> run -d` / `<engine> stop` |
| `quadlet` | `run_mode: quadlet` | systemd user services via podman quadlet |

```bash
ov config set run_mode direct   # Default
ov config set run_mode quadlet  # systemd integration
```

## Device Auto-Detection

By default, `ov shell`, `ov start`, `ov enable`, and `ov vm create` auto-detect available host devices and pass them through to the container. Use `--no-autodetect` to disable.

**Auto-detected devices:**
- NVIDIA GPU (via `nvidia-smi`) — passed as `--gpus all` (Docker) or `--device nvidia.com/gpu=all` (Podman)
- `/dev/dri/renderD*` — GPU render nodes
- `/dev/kvm` — KVM virtualization
- `/dev/vhost-net`, `/dev/vhost-vsock` — virtio networking
- `/dev/fuse` — FUSE filesystem
- `/dev/net/tun` — TUN/TAP networking
- `/dev/hwrng` — hardware RNG

| Flag | Behavior |
|------|----------|
| (default) | Auto-detect GPU + devices |
| `--no-autodetect` | Disable all device auto-detection |

Source: `ov/devices.go` (`DetectHostDevices`, `DetectGPU`).

## Shell Command

```bash
ov shell <image>                          # Interactive bash
ov shell <image> -w ~/project             # Mount workspace
ov shell <image> -c "nvidia-smi"          # Run command
ov shell <image> --tag 2026.46.1415       # Specific version
ov shell <image> --no-autodetect           # Disable device auto-detection
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
ov remove <image> --volumes       # Stop + cleanup + delete named volumes
ov remove <image> -e KEY=VALUE    # Set env vars for lifecycle hooks
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
| `engine.rootful` | `OV_ENGINE_ROOTFUL` | `false` | Use rootful podman (via podman machine) |
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

All runtime commands accept remote image references: `github.com/org/repo/image[@version]`.

```bash
ov shell github.com/org/repo/my-app               # Pull and run
ov shell github.com/org/repo/my-app@v1.0.0        # Specific version
ov enable github.com/org/repo/my-app --build       # Force local build
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

## Supervisord Service Management

Manage individual supervisord services inside a running container:

```bash
ov service status my-app               # Show status of all supervisord services
ov service start my-app traefik        # Start a specific service
ov service stop my-app traefik         # Stop a specific service
ov service restart my-app traefik      # Restart a specific service
ov service status my-app -i prod       # Named instance
```

The service name must match a `[program:<name>]` entry in the image's supervisord config. Available services are validated against the image's `org.overthinkos.supervisord` label.

Source: `ov/service.go`.

## Container Networking

All containers are connected to a shared `ov` network by default, enabling inter-container DNS resolution by container name. Override with the `network` field in images.yml (e.g., `network: host`).

Source: `ov/network.go` (`ResolveNetwork`).

## VM Management

For bootc images, `ov vm` commands manage virtual machines built from disk images:

```bash
ov vm create my-image                  # Create VM from disk image
ov vm create my-image --ram 8G --cpus 4  # Custom resources
ov vm create my-image --ssh-key auto   # Auto-detect SSH key (default)
ov vm create my-image --ssh-key generate  # Generate new SSH keypair
ov vm create my-image -i prod          # Named instance
ov vm start my-image                   # Start a stopped VM
ov vm stop my-image                    # Graceful shutdown
ov vm destroy my-image --disk          # Remove VM and delete disk
ov vm list -a                          # List all VMs
ov vm console my-image                 # Serial console
ov vm ssh my-image -p 2222 -l user     # SSH access
```

See `/overthink:deploy` for full VM configuration, backends, and libvirt XML injection.

## Lifecycle Hooks

Layers can declare lifecycle hooks that run on the host at specific points in the service lifecycle:

```yaml
# In layer.yml:
hooks:
  post_enable: |
    echo "Service enabled"
  pre_remove: |
    echo "About to remove service"
```

| Hook | When it runs |
|------|-------------|
| `post_enable` | After `ov enable` generates the quadlet and reloads systemd |
| `pre_remove` | Before `ov remove` stops and removes the service |

Hooks from multiple layers are concatenated in layer order. Hook scripts run on the host (not inside the container). The `ov remove -e KEY=VALUE` flag passes environment variables to hook scripts.

Source: `ov/hooks.go` (`CollectHooks`, `RunHook`).

## Cross-Engine Image Transfer

When `engine.build` differs from `engine.run`, images are automatically transferred between engines on demand.

| Function | Purpose |
|----------|---------|
| `LocalImageExists(engine, imageRef)` | Check if image exists in engine's local store |
| `TransferImage(srcEngine, dstEngine, imageRef)` | Bidirectional pipe: `<src> save <ref> \| <dst> load` |
| `EnsureImage(imageRef, rt)` | Transfer from build to run engine if needed, error if missing from both |

Transfer is called automatically by `ov shell`, `ov start`, `ov enable`, and `ov update`.

Source: `ov/transfer.go`.

## Browser Automation (ov browser)

`ov browser` commands connect to Chrome DevTools Protocol (CDP) on port 9222 inside running containers. Requires a Chrome layer with `port_relay: [9222]` and `--remote-allow-origins='*'` in the Chrome launch flags.

### Architecture

1. Resolves the container name from image + instance
2. Discovers the mapped port 9222 via `podman port` / `docker port`
3. HTTP API (`/json/list`, `/json/new?url=`) for list, open, and close operations
4. CDP WebSocket for interactive operations (click, type, eval, wait, text, html, screenshot, cdp)

### Commands

```bash
ov browser open <image> <url>                    # Open URL in new Chrome tab
ov browser list <image>                           # List all open tabs (id, title, url)
ov browser close <image> <tab-id>                 # Close a tab by ID
ov browser text <image> <tab-id>                  # Get page text content
ov browser html <image> <tab-id>                  # Get page HTML source
ov browser url <image> <tab-id>                   # Get page title and URL
ov browser screenshot <image> <tab-id> [file]     # Capture screenshot as PNG
ov browser click <image> <tab-id> <selector>      # Click element by CSS selector
ov browser type <image> <tab-id> <selector> <text>  # Type into input field
ov browser eval <image> <tab-id> <expression>     # Evaluate JavaScript expression
ov browser wait <image> <tab-id> <selector>       # Wait for element (--timeout 30s)
ov browser cdp <image> <tab-id> <method> [json]   # Send raw CDP command
```

All commands accept `-i INSTANCE` for multi-instance support.

### Example: Navigate and Extract Data

```bash
ov browser open my-app "https://example.com"
TAB=$(ov browser list my-app | head -1 | awk '{print $1}')
ov browser wait my-app $TAB 'h1'
ov browser text my-app $TAB
ov browser screenshot my-app $TAB page.png
```

Source: `ov/browser.go`, `ov/browser_cdp.go`.

## --tty Flag

`ov shell <image> --tty -c "command"` forces TTY allocation even when stdout is not a terminal. Uses `script(1)` from util-linux to provide a real PTY.

This is essential for automation tools (Claude Code, CI runners) where the process stdout is a pipe, not a terminal. Many interactive CLI commands (OAuth flows, prompts) require a TTY to function.

```bash
# Without --tty: "not a tty" errors from interactive commands
ov shell my-app -c "interactive-cli auth login"  # fails

# With --tty: script(1) wraps the command in a PTY
ov shell my-app --tty -c "interactive-cli auth login"  # works
```

Works for both new containers (`<engine> run`) and exec into running containers (`<engine> exec`).

## Port Relay

The `port_relay` field in layer.yml solves the problem of services that bind only to 127.0.0.1 inside the container (e.g., Chrome 146+ DevTools). Container port mappings only forward to interfaces visible from outside, so a loopback-only service is unreachable from the host.

```yaml
# In layer.yml:
port_relay:
  - 9222
```

How it works:
- socat binds to `eth0:<port>` and forwards to `127.0.0.1:<port>` (same port, different interface)
- A supervisord service is auto-generated with priority 1 (starts before other services)
- The socat layer is automatically added as a dependency

This makes loopback-only services accessible through normal podman/docker port mappings and `tailscale serve`.

Source: `ov/generate.go` (relay service generation), `ov/layers.go` (`PortRelayYAML`).

## OAuth Automation Example

Complete flow for deploying openclaw with Codex OAuth, using browser automation and TTY support:

```bash
# 1. Build and deploy with tailscale serve on all ports
ov build openclaw-sway-browser
ov enable openclaw-sway-browser
# Quadlet generates: tailscale serve --https=18789, --tcp=5900, --https=9222

# 2. Start OAuth process inside the running container
ov shell openclaw-sway-browser --tty -c \
  "openclaw models auth login --provider openai-codex --set-default" > /tmp/oauth.log &

# 3. Extract OAuth URL, open in container's Chrome via CDP
OAUTH_URL=$(grep -oP 'https://auth\.openai\.com\S+' /tmp/oauth.log)
ov browser open openclaw-sway-browser "$OAUTH_URL"

# 4. Wait for login page, click "Continue with Google"
TAB=$(ov browser list openclaw-sway-browser | grep openai | awk '{print $1}')
ov browser wait openclaw-sway-browser $TAB 'button[value="google"]'
ov browser click openclaw-sway-browser $TAB 'button[value="google"]'

# 5. Wait for consent page, click "Continue"
ov browser wait openclaw-sway-browser $TAB 'button[type="submit"]'
ov browser click openclaw-sway-browser $TAB 'button[type="submit"]'

# 6. OAuth callback hits localhost:1455 inside container
# Tokens saved to ~/.openclaw volume
# Default model: openai-codex/gpt-5.4
```

Key enablers:
- `--tty` provides PTY for interactive CLI commands from automation
- `port_relay` makes Chrome DevTools accessible from host through podman bridge networking
- `browser-open` + `BROWSER` env lets CLI tools open URLs in the running Chrome via CDP
- `tailscale serve --tcp=5900` exposes VNC for manual inspection if needed
- `shm_size: 1g` prevents Chrome from crashing due to /dev/shm exhaustion

## Cross-References

- `/overthink:build` -- Building images before running
- `/overthink:deploy` -- Quadlet and deployment details
- `/overthink:image` -- Image configuration (ports, volumes, aliases)

## When to Use This Skill

Use when the user asks about:

- Running containers or shells (`ov shell`, `ov start`)
- Service management (start, stop, status, logs)
- Device auto-detection and GPU passthrough
- Supervisord service management (`ov service`)
- Browser automation (`ov browser`)
- TTY allocation for automation (`--tty`)
- Port relay for loopback-only services
- Command aliases
- Runtime configuration
- Environment variable injection (`-e`, `--env-file`)
- Multi-instance containers (`--instance`)
- Remote image references
- Seeding bind mount data (`ov seed`)
- OAuth or interactive flows in containers
- "How do I run X in a container?"
