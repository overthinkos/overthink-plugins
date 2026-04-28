---
name: filebrowser
description: |
  FileBrowser Quantum web file manager on port 8080 with config-file-driven setup.
  Use when working with FileBrowser, web file management, or file browsing in containers.
---

# filebrowser -- Web file manager

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord` |
| Ports | 8080 |
| Volumes | `data` -> `~/.filebrowser/data`, `files` -> `~/.filebrowser/files` |
| Service | `filebrowser` (supervisord) |
| Install files | `tasks:`, `config.yaml` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `FILEBROWSER_CONFIG` | `/etc/filebrowser/config.yaml` |

## Aliases

| Name | Command |
|------|---------|
| `filebrowser` | `filebrowser` |

## Key Details

FileBrowser Quantum (gtsteffaniak fork, v1.2.4-stable) is config-file driven -- it uses
`/etc/filebrowser/config.yaml`, NOT CLI flags like classic filebrowser. The `FILEBROWSER_CONFIG`
environment variable points to the config file.

Default credentials: `admin` / `admin`. Change immediately after first login via the web UI.

Database and cache stored in the `data` volume (`~/.filebrowser/data/database.db`).
User-accessible files served from the `files` volume (`~/.filebrowser/files`).

The `files` volume is typically bind-mounted to a host directory at deploy time:

```bash
ov config filebrowser --bind files=~/Documents
ov config filebrowser --bind files=/mnt/nas/shared
```

## Usage

```yaml
# image.yml
filebrowser:
  base: fedora
  layers:
    - agent-forwarding
    - filebrowser
    - dbus
    - ov
  ports:
    - "8085:8080"
```

Tunnel config is in `deploy.yml`: `tunnel: {provider: tailscale, private: all}`. See `/ov:deploy`.

## Used In Images

- `/ov-images:filebrowser`

## Related Layers

- `/ov-layers:supervisord` -- process manager dependency
- `/ov-layers:traefik` -- similar pattern (binary download + config file + supervisord)
- `/ov-layers:dbus` -- D-Bus session bus (co-deployed in filebrowser image)
- `/ov-layers:ov` -- ov CLI (co-deployed in filebrowser image)

## Related Commands

- `/ov:config` -- deployment configuration, `--bind files=<path>` for file volume
- `/ov:start` -- start the service
- `/ov:shell` -- interactive access to the container

## When to Use This Skill

Use when the user asks about:

- FileBrowser setup in containers
- Web file management or file browsing
- Port 8080 file browser service
- `FILEBROWSER_CONFIG` configuration
- FileBrowser Quantum (gtsteffaniak fork)

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
