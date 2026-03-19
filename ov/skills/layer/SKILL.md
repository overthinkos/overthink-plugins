---
name: layer
description: |
  Layer authoring: creating, modifying, and understanding layer definitions.
  Use when working with layer.yml, root.yml, user.yml, pixi.toml, package.json,
  or Cargo.toml files under layers/.
---

# Layer - Layer Authoring

## Overview

A **layer** is a directory under `layers/<name>/` that installs a single concern. Layers are the building blocks of container images in overthink. Each layer can declare packages, dependencies, environment variables, ports, services, volumes, and command aliases.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Scaffold new layer | `ov new layer <name>` | Create layer directory with template files |
| List all layers | `ov list layers` | Show available layers from filesystem |
| List services | `ov list services` | Layers with `service` in layer.yml |
| List volumes | `ov list volumes` | Layers with `volumes` in layer.yml |
| List aliases | `ov list aliases` | Layers with `aliases` in layer.yml |
| Validate | `ov validate` | Check all layers and images |

## Install Files (processed in order)

| File | Runs as | Purpose |
|------|---------|---------|
| `layer.yml` `rpm`/`deb` | root | System packages declared in layer.yml |
| `root.yml` | root | Custom root install logic (Taskfile). Binary downloads, system config |
| `pixi.toml` / `pyproject.toml` / `environment.yml` | user | Python/conda packages. Multi-stage build. Only one per layer |
| `package.json` | user | npm packages -- installed globally via `npm install -g` |
| `Cargo.toml` | user | Rust crate -- built via `cargo install --path`. Requires `src/` directory |
| `user.yml` | user | Custom user install logic (Taskfile). Post-install config |

**Root vs User Rule:** System packages in `layer.yml` and `root.yml` run as root. Everything else runs as user. Pixi, npm, and cargo must never run as root.

## layer.yml Fields

| Field | Type | Purpose |
|-------|------|---------|
| `depends` | `[]string` | Layer dependencies. Resolved transitively, topologically sorted |
| `layers` | `[]string` | Compose other layers into this one. Layers with only `layers:` and no install files are valid (pure composition) |
| `env` | `map[string]string` | Environment variables. Merged across layers |
| `path_append` | `[]string` | Paths to append to `$PATH` |
| `ports` | `[]int` | Exposed ports (1-65535). Deduplicated across layers |
| `route` | `{host, port}` | Traefik reverse proxy route |
| `service` | multiline string | Supervisord `[program:<name>]` fragment |
| `rpm` | object | RPM package config: `packages`, `copr`, `repos`, `exclude`, `options` |
| `deb` | object | Debian package config: `packages` |
| `volumes` | `[]VolumeYAML` | Persistent named volumes (`name` + `path`) |
| `aliases` | `[]AliasYAML` | Host command aliases (`name` + `command`) |
| `security` | `SecurityConfig` | Container security: `privileged`, `cap_add`, `devices`, `security_opt`, `shm_size` |
| `port_relay` | `[]int` | Ports needing eth0->loopback socat relay (auto-adds socat dependency) |
| `hooks` | `HooksConfig` | Lifecycle hooks: `post_enable` (runs after `ov enable`), `pre_remove` (runs before `ov remove`) |
| `libvirt` | `[]string` | Raw libvirt XML snippets injected into VM domain XML after creation |

### port_relay

Ports needing an eth0->loopback socat relay inside the container. For services that bind only to 127.0.0.1 (like Chrome DevTools). Auto-adds socat dependency and generates a supervisord relay service.

```yaml
port_relay:
  - 9222
```

### Port Protocol Annotations

Ports support a protocol prefix for tailscale serve mode.

```yaml
ports:
  - 18789        # http (default) -> tailscale serve --https
  - tcp:5900     # tcp -> tailscale serve --tcp
  - 9222         # http (default)
```

### security.shm_size

Shared memory size. Largest value from all layers wins. Passed as `--shm-size` to podman/docker and `ShmSize=` in quadlet.

```yaml
security:
  shm_size: "1g"    # Chrome needs >64MB default
```

## Package Manager Sections

### RPM (`rpm:`)

```yaml
rpm:
  packages:
    - neovim
    - ripgrep
  copr:
    - owner/project       # Enabled before install, disabled after
  repos:
    - name: myrepo
      url: https://example.com/repo
      gpgkey: https://example.com/key.gpg
  exclude:
    - pattern-to-exclude
  options:
    - --setopt=tsflags=noscripts
```

### Deb (`deb:`)

```yaml
deb:
  packages:
    - neovim
    - ripgrep
```

## Dependencies

Layers declare dependencies via `depends` in layer.yml. The generator resolves transitively, topologically sorts, and pulls in missing dependencies automatically. Circular dependencies are a validation error. Layers already installed by a parent image are skipped.

```yaml
depends:
  - python
  - supervisord
```

## Environment Variables

```yaml
env:
  PIXI_CACHE_DIR: "~/.cache/pixi"
  MY_VAR: "value"

path_append:
  - "~/.pixi/bin"
  - "~/.local/bin"
```

`~` and `$HOME` are expanded to the resolved home directory at generation time. Setting `PATH` directly in `env` is a validation error -- use `path_append`. Later layers override earlier for the same key. `path_append` paths are accumulated across all layers.

Source: `ov/env.go` (`writeLayerEnv()`, `MergeEnvConfigs`, `ExpandEnvConfig`).

## Service Declaration

```yaml
service: |
  [program:myservice]
  command=/usr/bin/myservice --flag
  autostart=true
  autorestart=true
```

Requires `supervisord` layer in the dependency chain.

## Volume Declaration

```yaml
volumes:
  - name: data
    path: "~/.myapp"
```

Names must match `^[a-z0-9]+(-[a-z0-9]+)*$`. Docker/podman volume names become `ov-<image>-<name>`.

`CollectImageVolumes()` traverses the full image base chain (image -> base -> base's base), collecting volumes from all layers. Deduplicated by name (first declaration wins -- outermost image takes priority). Volumes are automatically mounted by `ov shell`, `ov start`, and `ov enable`.

Source: `ov/volumes.go`, `ov/layers.go` (`VolumeYAML`, `HasVolumes`, `Volumes()`).

## Security Declaration

```yaml
security:
  privileged: false
  cap_add:
    - SYS_PTRACE
  devices:
    - /dev/dri
  security_opt:
    - label:disable
```

Security settings are merged across layers: if any layer sets `privileged: true`, the result is privileged. `cap_add`, `devices`, and `security_opt` are unioned (deduplicated). Image-level `security:` in `images.yml` overrides `privileged` and appends to the other fields.

Source: `ov/security.go` (`CollectSecurity`, `SecurityArgs`).

## Cache Mounts

| Step type | Cache path | Options |
|-----------|------------|---------|
| `rpm.packages`, `root.yml` (rpm) | `/var/cache/libdnf5` | `sharing=locked` |
| `deb.packages`, `root.yml` (deb) | `/var/cache/apt` + `/var/lib/apt` | `sharing=locked` |
| `user.yml` | `/tmp/npm-cache` | `uid=<UID>,gid=<GID>` |
| `Cargo.toml` | `/tmp/cargo-cache` | `uid=<UID>,gid=<GID>` |
| pixi build stage | `/tmp/pixi-cache` + `/tmp/rattler-cache` | `uid=<UID>,gid=<GID>` |
| npm build stage | `/tmp/npm-cache` | `uid=<UID>,gid=<GID>` |

UID/GID in cache mounts are dynamic (from resolved image config, not hardcoded 1000). All non-root cache mounts use flat `/tmp/<tool>-cache` paths to avoid buildah permission issues with nested paths.

## Common Workflows

### Create a New Layer

```bash
ov new layer my-tool
# Edit layers/my-tool/layer.yml to add packages and dependencies
# Add root.yml for binary downloads, user.yml for post-install config
```

### Add Python Packages

Create `layers/my-tool/pixi.toml` with dependencies. The build system handles multi-stage builds automatically via the configured builder image.

### Add a Service

Add a `service` field to layer.yml with a supervisord program fragment. Add `supervisord` to `depends`.

## Style Guide

- Lowercase-hyphenated names for layers
- `root.yml` / `user.yml`: single `install` task, no parameters, idempotent
- System packages in `layer.yml` `rpm:`/`deb:` sections, not in root.yml
- Python in `pixi.toml`, npm in `package.json`, Rust in `Cargo.toml`
- Binary downloads in `root.yml`: detect arch with `uname -m`, map via `case`
- Never `pip install`, `conda install`, or `dnf install python3-*`
- Never `dnf clean all` in root.yml (cache mounts handle it)

## Cross-References

- `/ov:image` -- Adding layers to image definitions
- `/ov:build` -- Building images with layers

## When to Use This Skill

Use when the user asks about:

- Creating new layers or layer scaffolding
- Editing `layer.yml`, `root.yml`, `user.yml` files
- Adding packages (rpm, deb, pixi, npm, cargo)
- Layer dependencies and ordering
- Services, volumes, aliases, environment variables
- "How do I add X to the container?"
