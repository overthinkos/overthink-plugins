---
name: service
description: |
  MUST be invoked before any work involving: ov start/stop/enable/disable/status/logs/update/remove commands, supervisord service management, or container lifecycle.
---

# Service - Service Management

## Overview

Container service lifecycle management with two modes: **quadlet** (systemd user services via podman quadlet, always preferred) and **direct** (`<engine> run -d` / `<engine> stop`, fallback only for platforms without quadlet support). Also manages individual supervisord services inside running containers.

## Quick Reference

### Container Lifecycle

| Action | Command | Description |
|--------|---------|-------------|
| Start service | `ov start <image>` | Start as background service |
| Stop service | `ov stop <image>` | Stop running service |
| Enable quadlet | `ov enable <image>` | Generate .container file, daemon-reload |
| Disable quadlet | `ov disable <image>` | Disable auto-start |
| Service status | `ov status <image>` | Show service status |
| Service logs | `ov logs <image> -f` | Follow service logs |
| Update service | `ov update <image>` | Update image and restart |
| Remove service | `ov remove <image>` | Stop and remove service |
| Remove + volumes | `ov remove <image> --volumes` | Also delete named volumes |
| Seed data | `ov seed <image>` | Copy image data to empty bind mount dirs |

### Supervisord Services

| Action | Command | Description |
|--------|---------|-------------|
| Service status | `ov service status <image>` | Show all supervisord service status |
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
ov config set run_mode quadlet  # Recommended -- systemd integration
ov config set run_mode direct   # Fallback for platforms without quadlet
```

## Quadlet Mode

User-level systemd services via podman quadlet. No root required.

### Setup

```bash
ov config set run_mode quadlet
ov config set engine.run podman    # Required
loginctl enable-linger $USER       # Required for user services
```

### Workflow

```bash
ov enable my-app -w ~/project                          # Generate .container file
ov enable my-app -i prod -w ~/prod -e ENV=production   # Named instance with env
ov start my-app                    # systemctl --user start
ov status my-app                   # systemctl --user status
ov logs my-app -f                  # journalctl --user -u (follow)
ov update my-app                   # Re-transfer image, restart
ov stop my-app                     # systemctl --user stop
ov disable my-app                  # Disable auto-start
ov remove my-app                   # Stop + remove .container file
ov remove my-app --volumes         # Also remove named volumes
ov remove my-app -e KEY=VALUE      # Set env vars for lifecycle hooks
```

With `auto_enable=true`, `ov start` auto-generates the quadlet file on first run.

### Generated Files

- Path: `~/.config/containers/systemd/ov-<image>.container` (or `ov-<image>-<instance>.container`)
- Service name: `ov-<image>.service`
- Container name: `ov-<image>`
- Ports bound to configured `bind_address`
- Entrypoint: `supervisord -n -c /etc/supervisord.conf`
- Auto-restart on failure via `WantedBy=default.target`

### Environment Variables in Quadlet

- **`Environment=`** lines for CLI `-e` flags (inline in .container file)
- **`EnvironmentFile=`** directive for file-sourced vars (`--env-file`, workspace `.env`, or `env_file` in images.yml)

When `EnvironmentFile=` is used, only explicit CLI `-e` vars appear as inline `Environment=` to avoid duplication.

### Security in Quadlet

Layer and image-level security settings become `PodmanArgs=` in the quadlet file:

- `privileged: true` -> `PodmanArgs=--privileged`
- `cap_add` -> `PodmanArgs=--cap-add=<CAP>`
- `devices` -> `PodmanArgs=--device=<DEV>`
- `security_opt` -> `PodmanArgs=--security-opt=<OPT>`

Source: `ov/security.go`, `ov/quadlet.go`.

### Image Transfer

When `engine.build=docker`, `ov enable` auto-detects if the image is missing from podman and transfers via `docker save | podman load`. `ov update` re-transfers if needed.

## Direct Mode (Fallback)

Only use direct mode on platforms that don't support quadlet (e.g., macOS Docker Desktop).

```bash
ov start my-app                  # docker/podman run -d
ov start my-app -w ~/project     # With workspace mount
ov start my-app -e LOG=debug     # With env vars
ov status my-app                 # docker/podman inspect
ov logs my-app -f                # docker/podman logs -f
ov stop my-app                   # docker/podman stop
ov update my-app --build         # Rebuild, stop old, start new
ov remove my-app                 # Stop + remove container
ov remove my-app --volumes       # Also remove named volumes
```

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
| `post_enable` | After `ov enable` generates the quadlet and reloads systemd |
| `pre_remove` | Before `ov remove` stops and removes the service |

Hooks from multiple layers are concatenated in layer order. Scripts run on the host (not inside the container). Use `ov remove -e KEY=VALUE` to pass environment variables to hook scripts.

Source: `ov/hooks.go`.

## Multi-Instance Support

The `-i NAME` flag enables running multiple containers of the same image with separate state:

```bash
ov enable my-app -i prod -w ~/prod
ov enable my-app -i staging -w ~/staging
ov start my-app -i prod
ov status my-app -i staging
```

Instance naming affects:
- Container name: `ov-<image>` -> `ov-<image>-<instance>`
- Volume names: `ov-<image>-<name>` -> `ov-<image>-<instance>-<name>`
- Quadlet file: `ov-<image>.container` -> `ov-<image>-<instance>.container`
- Service name: `ov-<image>.service` -> `ov-<image>-<instance>.service`

Source: `ov/volumes.go` (`InstanceVolumes`), `ov/quadlet.go`.

## Seed Command

`ov seed <image>` copies image data into empty bind mount directories that override layer volumes.

```bash
ov seed my-app                 # Seed all empty bind mount dirs
ov seed my-app --tag v1.0.0    # From specific image version
```

Only seeds bind mounts that match layer volume names. Skips directories with existing data.

Source: `ov/seed.go`.

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

## Cross-References

- `/ov:shell` -- Interactive shells and exec into running containers
- `/ov:deploy` -- Quadlet generation details, tunnels, bind mounts
- `/ov:enc` -- Encrypted storage companion service
- `/ov:config` -- `run_mode`, `auto_enable`, `engine.run` settings

## When to Use This Skill

**MUST be invoked** when the task involves starting, stopping, enabling, or managing container services, supervisord service management, or container lifecycle. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** After `/ov:build` and `/ov:deploy`. This skill covers the runtime lifecycle.
Previous step: `/ov:deploy` (quadlet generation, tunnels). Next step: `/ov-images:<name>` (verification).
