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

Generated files: `~/.config/containers/systemd/ov-<image>.container`. Service name: `ov-<image>.service`. Container name: `ov-<image>`. Ports bound to configured `bind_address`. Entrypoint: `supervisord -n -c /etc/supervisord.conf`. Auto-restart on failure via `WantedBy=default.target`.

### Image Transfer

When `engine.build=docker`, `ov enable` auto-detects if the image is missing from podman and transfers via `docker save | podman load`. `ov update` re-transfers if needed and restarts the service.

Source: `ov/quadlet.go` (generation), `ov/commands.go` (command structs).

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

### Tailscale Serve (tailnet-private, default)

Exposes a port to your Tailscale network only. No FQDN needed -- Tailscale handles TLS automatically.

```yaml
tunnel: tailscale
# or expanded:
tunnel:
  provider: tailscale
  port: 2283
  https: 443          # default: 443
```

Allowed HTTPS ports for serve: 80, 443, 3000-10000, 4443, 5432, 6443, 8443.

### Tailscale Funnel (public internet)

Exposes a port to the public internet via Tailscale's edge network.

```yaml
tunnel:
  provider: tailscale
  funnel: true
  port: 8080
  https: 443    # must be 443, 8443, or 10000
```

### Cloudflare Tunnel

Routes traffic through Cloudflare's network. Requires `fqdn`.

```yaml
tunnel:
  provider: cloudflare
  port: 3001
  tunnel: my-tunnel    # optional, defaults to ov-<image>
fqdn: "app.example.com"
```

### Resolution

`tunnel` inherits from defaults (image -> defaults -> nil). The `port` field defaults to the first route port from layers if not specified. For tailscale, `https` defaults to 443.

Source: `ov/tunnel.go`, `ov/validate.go` (`validateTunnel`), `ov/quadlet.go` (systemd integration).

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

## Bind Mounts

Image-level host-path bind mounts declared in `images.yml` or `deploy.yml`. Two modes: plain (direct bind) and encrypted (gocryptfs-managed). Not inherited from defaults -- deployment-specific.

### Configuration

```yaml
bind_mounts:
  - name: data
    host: "~/data/myapp"        # host dir, required for plain mounts
    path: "~/.myapp"            # container path, ~ expanded to resolved home

  - name: secrets
    path: "~/.myapp/secrets"    # container path
    encrypted: true             # gocryptfs, host managed by ov
```

**Rules:**
- `encrypted: false` (default): `host` is required -- direct bind mount
- `encrypted: true`: `host` is forbidden -- ov manages cipher/plain dirs at `encrypted_storage_path`
- `path`: container mount path (required). `~`/`$HOME` expanded to the image's resolved home dir
- `host`: host mount path (plain mounts only). `~`/`$HOME` expanded to the user's actual home
- `name`: unique identifier, must match `^[a-z0-9]+(-[a-z0-9]+)*$`
- Bind mount names must not collide with layer volume names (same namespace)

### Bind Mount / Volume Override

When a bind mount has the same name as a layer volume, the bind mount **overrides** the volume. The named volume is not created -- the bind mount is used instead. This is an informational note on stderr. This allows deploy.yml to replace layer-declared volumes with encrypted bind mounts.

### Encrypted Bind Mounts

```bash
ov crypto init my-app [--volume NAME]     # Initialize gocryptfs cipher dirs
ov crypto mount my-app [--volume NAME]    # Mount (prompts for password once)
ov crypto unmount my-app [--volume NAME]  # Unmount
ov crypto status my-app                   # Show status
```

Storage layout: `~/.local/share/ov/encrypted/ov-<image>-<name>/cipher/` and `plain/`. Override path: `OV_ENCRYPTED_STORAGE_PATH`.

### Single Password

When an image has multiple encrypted bind mounts, `ov crypto init`, `ov crypto mount`, and the generated crypto systemd unit all use `systemd-ask-password --id=ov-<image>` to cache the passphrase in the kernel keyring. Prompted once and reused for all volumes.

### Integration

- **`ov shell`/`ov start` (direct)**: resolves bind mounts, verifies plain dirs exist and encrypted volumes are mounted, appends `-v <host>:<container>` flags
- **`ov enable` (quadlet)**: plain mounts as `Volume=` lines; encrypted mounts generate a companion `ov-<image>-crypto.service` with `Requires=`/`After=`
- **`ov remove`**: removes companion crypto service file
- **`ov inspect --format bind_mounts`**: outputs `NAME\tHOST\tPATH\tENCRYPTED`

Source: `ov/crypto.go`, `ov/validate.go` (`validateBindMounts`).

## Cross-References

- `/overthink:build` -- Building images before deployment
- `/overthink:run` -- Runtime commands (start, stop, shell)
- `/overthink:image` -- Image configuration

## When to Use This Skill

Use when the user asks about:

- Quadlet/systemd service deployment
- Bootc disk images (ISO, QCOW2, RAW)
- Pushing images to registries
- Tunnel configuration (Tailscale, Cloudflare)
- Encrypted storage and bind mounts
- Deploy overlays
- "How do I deploy X to production?"
