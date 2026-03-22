---
name: layer
description: |
  MUST be invoked before any work involving: layer authoring, layer.yml, root.yml, user.yml, pixi.toml, package.json, Cargo.toml, or any file under layers/.
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

**Auto-detection:** The build system scans each layer directory for these files. `package.json`, `pixi.toml`, `pyproject.toml`, `environment.yml`, and `Cargo.toml` trigger automatic multi-stage builds -- no manual install commands needed. Only use `root.yml`/`user.yml` for things that can't be expressed as package manifests (binary downloads, `go install`, post-install configuration).

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

Ports support a protocol prefix that controls EXPOSE format, tailscale serve mode, and port mapping.

```yaml
ports:
  - 18789        # http (default) -> EXPOSE 18789, tailscale serve --https
  - tcp:5900     # tcp -> EXPOSE 5900, tailscale serve --tcp
  - udp:47998    # udp -> EXPOSE 47998/udp, -p 47998:47998/udp, not tunneled
  - 9222         # http (default)
```

UDP ports generate `EXPOSE <port>/udp` in Containerfiles and `-p host:container/udp` in port mappings. Tailscale serve and Cloudflare tunnels do not support UDP — a warning is printed when UDP ports are encountered in tunnel configs.

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

## depends vs layers

`depends` and `layers` serve different purposes:

| | `depends` | `layers` |
|---|---|---|
| Purpose | Prerequisite ordering | Group composition |
| Effect | Ensures dependency is installed first | Splices layers into this layer's position |
| Transitive | Yes -- pulls in all sub-dependencies | Yes -- recursively expands nested `layers:` |
| Typical use | Runtime/build prerequisites | Metalayers, layer bundles |

**Examples:**

- Go tool layer -- needs Go compiler as prerequisite: `depends: [golang]`
- Python package layer -- needs Python runtime (not pixi!): `depends: [python]`
- Metalayer -- groups existing layers, no install logic: `layers: [openclaw, chrome, claude-code]`

**Common mistake:** Using `depends: [pixi]` when you mean `depends: [python]`. The `pixi` layer installs the pixi binary (build tool). The `python` layer installs Python via pixi. Your layer needs Python, not pixi.

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

Create `pixi.toml` in the layer directory. The build system handles multi-stage builds automatically. For packages not on conda-forge, use `[pypi-dependencies]`.

**layer.yml must depend on `python`, not `pixi`:**

```yaml
depends:
  - python    # Runtime dependency (Python interpreter)
              # NOT pixi (build tool, only in builder image)
```

Never use `pip install`, `conda install`, `pixi global install`, or `uv tool install` in `user.yml`. Always use `pixi.toml`.

### Add npm Packages

Create `package.json` in the layer directory. The build system runs `npm install -g` automatically in a multi-stage build.

```json
{
  "name": "my-tool-layer",
  "version": "1.0.0",
  "dependencies": {
    "my-npm-package": "*"
  }
}
```

**layer.yml must depend on `nodejs`:**

```yaml
depends:
  - nodejs
```

Do not create `root.yml` with `npm install -g` commands -- `package.json` handles this automatically.

### Add Go Packages

Use `user.yml` with `go install` (Go has no declarative manifest for global installs).

**layer.yml:**

```yaml
depends:
  - golang

env:
  GOPATH: "~/go"

path_append:
  - "~/go/bin"
```

**cgo dependencies:** If the Go package uses C libraries (cgo), add required `-devel` packages to `rpm:`. Check build errors for `pkg-config` failures (e.g., `alsa-lib-devel` for audio). Always run `go clean -cache` at the end to reduce image size.

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

**MUST be invoked** when the task involves layer authoring, layer.yml, root.yml, user.yml, pixi.toml, package.json, Cargo.toml, or any file under layers/. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Pre-build. Author layers before adding to images. See also `/ov:image` (image composition), `/ov:build` (building).
