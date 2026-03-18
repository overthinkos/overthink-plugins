---
name: service
description: |
  Service management: ov start/stop/enable/disable/status/logs/update/remove
  and ov service start/stop/restart/status for supervisord services.
  Use when managing container lifecycle or supervisord services.
---

# Service - Service Management

## Overview

Container service lifecycle management with two modes: **direct** (`<engine> run -d` / `<engine> stop`) and **quadlet** (systemd user services via podman quadlet). Also manages individual supervisord services inside running containers.

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
| `direct` | `run_mode: direct` | `<engine> run -d` / `<engine> stop` |
| `quadlet` | `run_mode: quadlet` | systemd user services via podman quadlet |

```bash
ov config set run_mode direct   # Default
ov config set run_mode quadlet  # systemd integration
```

## Direct Mode

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

## Cross-References

- `/overthink:shell` -- Interactive shells and exec into running containers
- `/overthink:deploy` -- Quadlet generation details, tunnels, bind mounts
- `/overthink:crypto` -- Encrypted storage companion service
- `/overthink:config` -- `run_mode`, `auto_enable`, `engine.run` settings

## When to Use This Skill

Use when the user asks about:

- Starting/stopping containers (`ov start`, `ov stop`)
- Quadlet systemd services (`ov enable`, `ov disable`)
- Service status and logs (`ov status`, `ov logs`)
- Updating or removing services (`ov update`, `ov remove`)
- Supervisord service management (`ov service`)
- Lifecycle hooks
- Multi-instance containers
- "How do I start/manage a service?"
