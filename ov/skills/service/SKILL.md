---
name: service
description: |
  MUST be invoked before any work involving: ov start/stop/status/logs/update/remove commands, ov config (deployment), init system service management, or container lifecycle.
---

# Service - Service Management

## Schema v3 terminology note

The schema-v3 rename settled on **"pod"** as the user-visible term for a single-container deployment (matches podman's vocabulary and the `target: pod` deploy-yml value). Internally, the Go struct is `PodDeployTarget` (formerly `ContainerDeployTarget`) and the file is `ov/deploy_target_pod.go`. This skill's body still uses the word "container" in many places because it's also the generic runtime artifact — read "container" as the runtime concept and "pod" as the schema-v3 target/deployment kind.

`ov start/stop/status/logs/shell` operate on named pod deployments (the unit a user cares about); the underlying runtime is podman/docker (containers), managed via systemd user quadlet.

## Overview

Container service lifecycle management with two modes: **quadlet** (systemd user services via podman quadlet, always preferred) and **direct** (`<engine> run -d` / `<engine> stop`, fallback only for platforms without quadlet support). Also manages individual init system services inside running containers (supervisord, systemd, etc. -- configured via build.yml `init:` section).

## Quick Reference

### Container Lifecycle

| Action | Command | Description |
|--------|---------|-------------|
| Start service | `ov start <image>` | Start as background service |
| Stop service | `ov stop <image>` | Stop running service |
| Configure deployment | `ov config <image>` | Generate .container file, daemon-reload |
| Remove deployment | `ov config remove <image>` | Remove deployment configuration |
| Service status | `ov status [<image>]` | Structured status table (IMAGE, STATUS, PORTS, DEVICES, TOOLS) |
| All services | `ov status --all` | Include stopped/enabled services in listing |
| Detailed status | `ov status <image>` | Detailed key-value view with live tool probes |
| JSON output | `ov status --json` | Machine-readable JSON output |
| Service logs | `ov logs <image> -f` | Follow service logs |
| Update service | `ov update <image>` | Update image and restart |
| Remove service | `ov remove <image>` | Stop, remove service + deploy.yml entry |
| Purge (+ volumes) | `ov remove <image> --purge` | Also delete named volumes |
| Remove keep config | `ov remove <image> --keep-deploy` | Remove service but keep deploy.yml entry |

### Supervisord Services

| Action | Command | Description |
|--------|---------|-------------|
| Service status | `ov service status <image>` | Show all service status |
| Start service | `ov service start <image> <svc>` | Start a specific service |
| Stop service | `ov service stop <image> <svc>` | Stop a specific service |
| Restart service | `ov service restart <image> <svc>` | Restart a specific service |

All commands accept `-i INSTANCE` for multi-instance support.

## Run Modes

| Mode | Config | How it works |
|------|--------|-------------|
| `quadlet` | `run_mode: quadlet` | systemd user services via podman quadlet (always preferred) |
| `direct` | `run_mode: direct` | `<engine> run -d` / `<engine> stop` (fallback only) |

```bash
ov settings set run_mode quadlet  # Recommended -- systemd integration
ov settings set run_mode direct   # Fallback for platforms without quadlet
```

## Quadlet Mode

User-level systemd services via podman quadlet. No root required.

### Setup

```bash
ov settings set run_mode quadlet
ov settings set engine.run podman    # Required
loginctl enable-linger $USER         # Required for user services
```

### Workflow

```bash
ov config my-app --bind workspace=~/project              # Generate .container file
ov config my-app -i prod --bind workspace=~/prod -e ENV=production   # Named instance
ov start my-app                    # systemctl --user start (config required first)
ov status my-app                   # systemctl --user status
ov logs my-app -f                  # journalctl --user -u (follow)
ov update my-app                   # Re-transfer image, restart
ov stop my-app                     # systemctl --user stop
ov config remove my-app            # Remove deployment configuration
ov remove my-app                   # Stop + remove .container + deploy.yml entry
ov remove my-app --purge           # Also remove named volumes
ov remove my-app --keep-deploy     # Remove service but keep deploy.yml for re-config
ov remove my-app -e KEY=VALUE      # Set env vars for lifecycle hooks
```

`ov config` must be run before `ov start` in quadlet mode. If the quadlet file doesn't exist, `ov start` fails with: "not configured; run 'ov config <image>' first".

### Generated Files

- Path: `~/.config/containers/systemd/ov-<image>.container` (or `ov-<image>-<instance>.container`)
- Service name: `ov-<image>.service`
- Container name: `ov-<image>`
- Ports bound to configured `bind_address`
- Entrypoint: determined by build.yml `init:` section config (e.g., `supervisord -n -c /etc/supervisord.conf` for supervisord, `sleep infinity` if no init system)
- Auto-restart on failure via `WantedBy=default.target` (encrypted services with Secret Service backend include `ExecStartPre=ov config mount` + `TimeoutStartSec=0` for keyring wait; KeePass/no backend omit `WantedBy` — require `ov start`)
- `ov image validate` enforces: images with init system layers MUST include the required dependency layer (defined by build.yml `init:` section `depends_layer`)
- `Secret=ov-<image>-<name>,target=/run/secrets/<name>` for each layer-declared secret (Podman only)

### Container Secrets

When image labels declare secrets (from `layer.yml` `secrets` field), `ov config` provisions them:
1. Checks if Podman secret already exists — **if so, keeps it** (never overwrites)
2. If missing: resolves secret values from the credential store (env var > keyring > config file)
3. Creates Podman secrets via `podman secret create ov-<image>-<name>`
4. Generates `Secret=` directives in the quadlet file
5. Secrets are mounted at `/run/secrets/<name>` inside the container

**Provisioning is idempotent** — existing secrets are never overwritten. This prevents breaking stateful services (e.g., PostgreSQL) that store their own copy of the password. To force re-provisioning: `podman secret rm <name> && ov config setup <image>`.

For Docker, secrets fall back to `Environment=` injection with a security warning.

### Environment Variables in Quadlet

- **`Environment=`** lines for CLI `-e` flags (inline in .container file)
- **`EnvironmentFile=`** directive for file-sourced vars (`--env-file`, workspace `.env`, or `env_file` in deploy.yml)

When `EnvironmentFile=` is used, only explicit CLI `-e` vars appear as inline `Environment=` to avoid duplication.

### Security in Quadlet

Layer and image-level security settings become `PodmanArgs=` in the quadlet file:

- `privileged: true` -> `PodmanArgs=--privileged`
- `cap_add` -> `PodmanArgs=--cap-add=<CAP>`
- `devices` -> `PodmanArgs=--device=<DEV>`
- `security_opt` -> `PodmanArgs=--security-opt=<OPT>`

Source: `ov/security.go`, `ov/quadlet.go`.

### Image Transfer

When `engine.build=docker`, `ov config` auto-detects if the image is missing from podman and transfers via `docker save | podman load`. `ov update` re-transfers if needed.

## Direct Mode (Fallback)

Only use direct mode on platforms that don't support quadlet (e.g., macOS Docker Desktop).

```bash
ov start my-app                  # docker/podman run -d
ov start my-app --bind workspace=~/project  # With volume binding
ov start my-app -e LOG=debug     # With env vars
ov status my-app                 # docker/podman inspect
ov logs my-app -f                # docker/podman logs -f
ov stop my-app                   # docker/podman stop
ov update my-app --build         # Rebuild, stop old, start new
ov remove my-app                 # Stop + remove container
ov remove my-app --purge         # Also remove named volumes
```

## Init System Service Management

Manage individual services inside a running container (uses the init system configured via build.yml `init:` section — supervisord, systemd, etc.):

```bash
ov service status my-app               # Show status of all services
ov service start my-app traefik        # Start a specific service
ov service stop my-app traefik         # Stop a specific service
ov service restart my-app traefik      # Restart a specific service
ov service status my-app -i prod       # Named instance
```

The service name must match an entry in the image's init system config. Available services are validated against the image's `org.overthinkos.services.<init>` label (e.g., `org.overthinkos.services.supervisord`). The management tool and commands are defined in build.yml `init:` section.

Source: `ov/service.go`.

## Lifecycle Hooks

Layers can declare hooks that run on the host at specific points:

```yaml
# In layer.yml:
hooks:
  post_enable: |
    echo "Service enabled for $OV_IMAGE"
  pre_remove: |
    echo "Cleaning up before removal"
```

| Hook | When it runs |
|------|-------------|
| `post_enable` | After `ov config` generates the quadlet and reloads systemd |
| `pre_remove` | Before `ov remove` stops and removes the service |

Hooks from multiple layers are concatenated in layer order. Scripts run on the host (not inside the container). Use `ov remove -e KEY=VALUE` to pass environment variables to hook scripts.

Source: `ov/hooks.go`.

## Multi-Instance Support

The `-i NAME` flag enables running multiple containers of the same image with separate state:

```bash
ov config my-app -i prod --bind workspace=~/prod
ov config my-app -i staging --bind workspace=~/staging
ov start my-app -i prod
ov status my-app -i staging
```

Instance naming affects:
- Container name: `ov-<image>` -> `ov-<image>-<instance>`
- Volume names: `ov-<image>-<name>` -> `ov-<image>-<instance>-<name>`
- Quadlet file: `ov-<image>.container` -> `ov-<image>-<instance>.container`
- Service name: `ov-<image>.service` -> `ov-<image>-<instance>.service`

Source: `ov/volumes.go` (`InstanceVolumes`), `ov/quadlet.go`.

## Data Provisioning

Data from data layers is automatically provisioned into bind-backed volumes during `ov config` and synced during `ov update`. See `/ov:config` for `--seed`/`--force-seed`/`--data-from` flags. Source: `ov/data.go`.

## Troubleshooting

### Port Already in Use

If `ov start` fails with `bind: address already in use`, another container or host process is using the port. Find the conflict:

```bash
ss -tlnp | grep <port>           # Find what's listening
podman ps --format '{{.Names}} {{.Ports}}' | grep <port>  # Find container
```

Stop the conflicting container before starting. Common conflicts: standalone `ollama` container on 11434, existing VNC on 5900.

### Service Crash-Looping

If `ov service status` shows a service cycling STARTING/STOPPED, run it manually to see the error:

```bash
ov shell <image> -c "supervisorctl stop <service>"
ov shell <image> -c "<service-command>"
```

## Status Output

`ov status` shows a structured table of all running ov containers:

```
IMAGE                       STATUS   PORTS                      DEVICES  TOOLS
openclaw-sway-browser       running  5900,8000,8080,18789       dri,gpu  supervisord:ok,cdp:9222,dbus,ov,sway,vnc:5900,wl
ollama                      running  11434                      gpu      supervisord:ok,dbus,ov
jupyter                     stopped  8888                       -        -
```

- **PORTS**: Host-side port numbers, sorted numerically
- **DEVICES**: Compact tokens (`gpu`, `dri`, `kvm`, `fuse`), sorted alphabetically
- **TOOLS**: Live-probed desktop automation tools with actual host ports, sorted alphabetically. Port-based tools show `name:port` (e.g., `cdp:9222`). Socket-based tools show just the name (e.g., `sway`). Non-running containers show `-`.

### Tool Probes

For running containers, `ov status` probes all tools concurrently:

| Tool | Probe | What it checks |
|------|-------|---------------|
| supervisord | `supervisorctl status` | Process manager health, service count (e.g., `8/8 running`) |
| cdp | HTTP GET `:9222/json` | Chrome DevTools Protocol |
| vnc | TCP + RFB handshake on `:5900` | VNC server (wayvnc) |
| sway | Sway IPC socket | Sway compositor |
| wl | `command -v grim wtype wlrctl` | Wayland tools |
| dbus | `pgrep -x dbus-daemon` | D-Bus session bus + notification daemon (swaync/mako/dunst) |
| ov | `which ov` + `ov version` | In-container ov binary + CalVer version |

Each tool also has its own `status` subcommand: `ov eval cdp status`, `ov eval vnc status`, `ov eval wl sway status`, `ov eval wl status`.

**Note:** `supervisorctl status` exits with code 3 when any service isn't RUNNING (e.g., FATAL, STOPPED). The probe correctly handles this — it parses the output regardless of exit code, only reporting `unreachable` when supervisord is truly not responding.

### Single Image Detail

`ov status <image>` shows a detailed key-value view:

```
Image:     openclaw-sway-browser
Status:    running (Up 3 days)
Container: ov-openclaw-sway-browser
Mode:      quadlet
Ports:     5900:5900, 8000:8000, 8080:8080, 18789:18789
Devices:   nvidia.com/gpu=all, /dev/dri/renderD128
Tools:     supervisord (ok), cdp:9222 (ok), dbus (ok), ov (ok), sway (ok), vnc:5900 (ok), wl (ok)
Volumes:   ov-openclaw-sway-browser-data -> ~/.openclaw
Workspace: ~/projects
Network:   host
Tunnel:    tailscale (all ports)
```

### JSON Output

`ov status --json` and `ov status <image> --json` emit structured JSON for scripting.

Source: `ov/status.go`.

## Cross-References

- `/ov:pull` -- Prerequisite: fetch the image into local storage; handles remote refs (`@github.com/...`) and the `ErrImageNotLocal` recovery path

- `/ov:shell` -- Interactive shells and exec into running containers
- `/ov:deploy` -- Quadlet generation details, tunnels, volume backing
- `/ov:enc` -- Encrypted storage (mounted inline by ov start)
- `/ov:config` -- `run_mode`, `auto_enable`, `engine.run` settings
- `/ov:cdp` -- CDP status subcommand (`ov eval cdp status`)
- `/ov:vnc` -- VNC status subcommand (`ov eval vnc status`)
- `/ov:wl` -- Desktop automation + sway subgroup (`ov eval wl sway status`)
- `/ov:wl` -- WL status subcommand (`ov eval wl status`)

## When to Use This Skill

**MUST be invoked** when the task involves starting, stopping, configuring, or managing container services, init system service management, or container lifecycle. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** After `/ov:build` and `/ov:deploy`. This skill covers the runtime lifecycle.
Previous step: `/ov:deploy` (quadlet generation, tunnels). Next step: `/ov-images:<name>` (verification).
