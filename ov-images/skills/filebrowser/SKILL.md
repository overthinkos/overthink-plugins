---
name: filebrowser
description: |
  FileBrowser Quantum web file manager with Tailscale tunnel.
  MUST be invoked before building, deploying, configuring, or troubleshooting the filebrowser image.
---

# filebrowser

Web file manager accessible via Tailscale private tunnel.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, filebrowser, dbus, ov |
| Platforms | linux/amd64 |
| Ports | 8085:8080 |
| Tunnel | tailscale (private: all) |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (base)
2. `pixi` -> `python` -> `supervisord` (transitive)
3. `agent-forwarding` -- SSH/GPG agent forwarding
4. `filebrowser` -- web file manager, data + files volumes
5. `dbus` -- D-Bus session bus
6. `ov` -- ov CLI inside container

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 8085 (host) -> 8080 (container) | FileBrowser | HTTP |

## Volumes

| Name | Path | Purpose |
|------|------|---------|
| data | ~/.filebrowser/data | Database (SQLite) and cache |
| files | ~/.filebrowser/files | User-accessible files (bind-mount target) |

## Quick Start

```bash
ov image build filebrowser
ov config filebrowser --bind files=~/Documents
ov start filebrowser
# Access at http://localhost:8085
# Tailscale: https://<hostname>:8085
# Default login: admin / admin
```

## Bind Mount Examples

```bash
# Browse home directory
ov config filebrowser --bind files=~/

# Browse NAS mount
ov config filebrowser --bind files=/mnt/nas/shared

# Browse specific project
ov config filebrowser --bind files=~/projects
```

## Host Alias

```bash
ov alias install filebrowser
# Now: filebrowser  (runs inside the container)
```

## Key Layers

- `/ov-layers:filebrowser` -- FileBrowser Quantum service, config, volumes
- `/ov-layers:agent-forwarding` -- SSH/GPG agent forwarding
- `/ov-layers:dbus` -- D-Bus session bus
- `/ov-layers:ov` -- ov CLI for in-container management

## Related Images

- `/ov-images:fedora` -- parent base image
- `/ov-images:ollama` -- similar simple service pattern

## Verification

After `ov start`:
- `ov status filebrowser` -- container running
- `curl -s http://localhost:8085` -- web UI responds (HTTP 200)
- `curl -s http://localhost:8085/health` -- `{"message":"ok"}`
- Tailscale: `https://<hostname>:8085` from any tailnet device

## Test Coverage

Latest `ov test filebrowser` run: **24 passed, 0 failed, 0 skipped**.
All tests embedded in the `org.overthinkos.tests` OCI label, covering
agent-forwarding prerequisites (gpg, ssh, direnv binaries), supervisord
+ dbus daemon + ov binary presence, filebrowser binary + config, and
deploy-scope: service up, port reachable on `127.0.0.1:${HOST_PORT:8080}`,
HTTP 200 on `/`, data volume mounted.

See `/ov:test` for the framework and author-facing gotchas.

## Related Skills

- `/ov-layers:filebrowser` — layer authoring
- `/ov:test` — declarative testing framework
- `/ov:config` — deploy-mode setup with volume backing and tunnels
- `/ov:build` — LABELs-at-end cache efficiency

## When to Use This Skill

**MUST be invoked** when the task involves the filebrowser image, web file management deployment, or the FileBrowser Quantum service. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
