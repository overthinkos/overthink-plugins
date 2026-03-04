---
name: deploy
description: |
  Deployment: quadlet systemd services, bootc disk images (ISO/QCOW2/RAW),
  registry push, tunnels, encrypted storage, and deploy overlays.
  Use for production deployment workflows.
---

# Deploy - Deployment

## Overview

Overthink supports multiple deployment modes: quadlet systemd user services for persistent containers, bootc disk images for bare-metal/VM installs, registry push for CI/CD, tunnel configuration for external access, and encrypted bind mounts for sensitive data.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Enable quadlet | `ov enable <image>` | Generate .container file, daemon-reload |
| Disable quadlet | `ov disable <image>` | Disable auto-start |
| Build ISO | `task build:iso -- <image>` | Bootc ISO installer |
| Build QCOW2 | `task build:qcow2 -- <image>` | VM disk image |
| Build RAW | `task build:raw -- <image>` | Raw disk image |
| Run VM | `task run:vm -- <image>` | Run QCOW2 in QEMU |
| Push to registry | `task build:push` | Multi-platform push |
| Init encryption | `ov crypto init <image>` | Initialize gocryptfs dirs |
| Mount encrypted | `ov crypto mount <image>` | Mount encrypted volumes |
| Unmount encrypted | `ov crypto unmount <image>` | Unmount encrypted volumes |

## Quadlet Services

User-level systemd services via podman quadlet. No root required.

### Setup

```bash
ov config set run_mode quadlet
ov config set engine.run podman
loginctl enable-linger $USER    # Required for user services
```

### Workflow

```bash
ov enable my-app -w ~/project    # Generate .container file, transfer image
ov start my-app                  # systemctl --user start
ov status my-app                 # systemctl --user status
ov logs my-app -f                # journalctl --user -u (follow)
ov update my-app                 # Re-transfer image, restart
ov stop my-app                   # systemctl --user stop
ov disable my-app                # Disable auto-start
ov remove my-app                 # Stop + remove .container file
```

With `auto_enable=true`, `ov start` auto-generates the quadlet file on first run.

Generated files: `~/.config/containers/systemd/ov-<image>.container`

## Bootc Disk Images

For images with `bootc: true` in images.yml. Requires `podman` (rootful).

```bash
task build:iso -- bazzite-ai          # ISO installer
task build:qcow2 -- bazzite-ai       # VM disk image
task build:raw -- bazzite-ai          # Raw disk for dd

task run:vm -- bazzite-ai             # Run QCOW2 in QEMU
```

Config files: `config/disk.toml` (QCOW2/RAW), `config/iso-gnome.toml` (ISO/Anaconda).

## Tunnel Configuration

Expose services outside the container host via tunnels.

### Tailscale Serve (tailnet-private)

```yaml
tunnel: tailscale
# or expanded:
tunnel:
  provider: tailscale
  port: 2283
  https: 443
```

### Tailscale Funnel (public internet)

```yaml
tunnel:
  provider: tailscale
  funnel: true
  port: 8080
  https: 443    # must be 443, 8443, or 10000
```

### Cloudflare Tunnel

```yaml
tunnel:
  provider: cloudflare
  port: 3001
  tunnel: my-tunnel
fqdn: "app.example.com"
```

## Deploy Overlay

Per-machine deployment overrides in `~/.config/ov/deploy.yml` (not checked into git):

```yaml
images:
  my-app:
    tunnel:
      provider: cloudflare
      port: 2283
    fqdn: "app.example.com"
    bind_mounts:
      - name: secrets
        path: "~/.myapp/secrets"
        encrypted: true
    ports:
      - "2283:2283"
```

Only deployment/runtime fields allowed: `tunnel`, `fqdn`, `acme_email`, `bind_mounts`, `ports`.

## Encrypted Bind Mounts

```yaml
# In images.yml or deploy.yml
bind_mounts:
  - name: secrets
    path: "~/.myapp/secrets"
    encrypted: true
```

```bash
ov crypto init my-app       # Initialize gocryptfs cipher dirs
ov crypto mount my-app      # Mount (prompts for password once)
ov crypto unmount my-app    # Unmount
ov crypto status my-app     # Show mount status
```

Storage: `~/.local/share/ov/encrypted/ov-<image>-<name>/cipher/` and `plain/`.

## Cross-References

- `/overthink:build` -- Building images before deployment
- `/overthink:run` -- Runtime commands (start, stop, shell)
- `/overthink:image` -- Image configuration
- Source: `ov/quadlet.go`, `ov/crypto.go`, `ov/deploy.go`, `ov/tunnel.go`

## When to Use This Skill

Use when the user asks about:

- Quadlet/systemd service deployment
- Bootc disk images (ISO, QCOW2, RAW)
- Pushing images to registries
- Tunnel configuration (Tailscale, Cloudflare)
- Encrypted storage and bind mounts
- Deploy overlays
- "How do I deploy X to production?"
