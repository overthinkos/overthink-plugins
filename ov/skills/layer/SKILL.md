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
| `root.yml` | root | Custom root install logic (Taskfile). Tag-based dispatch: `all:` + `rpm:`/`pac:`/`fedora:` tasks |
| `pixi.toml` / `pyproject.toml` / `environment.yml` | user | Python/conda packages. Multi-stage build. Only one per layer |
| `package.json` | user | npm packages -- installed globally via `npm install -g` |
| `Cargo.toml` | user | Rust crate -- built via `cargo install --path`. Requires `src/` directory |
| `build.sh` | user | Optional post-install script for pixi layers. Runs in pixi builder stage after `pixi install`. For build-time logic that can't be expressed in pixi.toml (C extension compilation, npm builds, binary patching). Builder image has gcc, nodejs, cmake, etc. |
| `user.yml` | user | Custom user install logic (Taskfile). Same tag-based dispatch as root.yml |

**Root vs User Rule:** System packages in `layer.yml` and `root.yml` run as root. Everything else runs as user. Pixi, npm, and cargo must never run as root.

**Auto-detection:** The build system scans each layer directory for these files. `package.json`, `pixi.toml`, `pyproject.toml`, `environment.yml`, and `Cargo.toml` trigger automatic multi-stage builds -- no manual install commands needed. Only use `root.yml`/`user.yml` for things that can't be expressed as package manifests (binary downloads, `go install`, post-install configuration).

## layer.yml Fields

| Field | Type | Purpose |
|-------|------|---------|
| `version` | `string` | CalVer version (`YYYY.DDD.HHMM`) of this layer definition. Set manually by author |
| `status` | `string` | `working`, `testing`, or `broken`. Default: `testing` (empty = testing) |
| `info` | `string` | Free-form description of what works/doesn't. Recommended when status is `testing` or `broken` |
| `depends` | `[]string` | Layer dependencies. Resolved transitively, topologically sorted |
| `layers` | `[]string` | Compose other layers into this one. Layers with only `layers:` and no install files are valid (pure composition) |
| `env` | `map[string]string` | Environment variables. Merged across layers |
| `path_append` | `[]string` | Paths to append to `$PATH` |
| `ports` | `[]int` | Exposed ports (1-65535). Deduplicated across layers |
| `route` | `{host, port}` | Traefik reverse proxy route |
| `service` | multiline string | Supervisord `[program:<name>]` fragment |
| `rpm` | object | RPM package config: `packages`, `copr`, `repos`, `exclude`, `options` |
| `deb` | object | Debian package config: `packages` |
| `pac` | object | Pacman package config: `packages`, `repos`, `options` |
| `aur` | object | AUR package config: `packages`, `options` (multi-stage build via yay) |
| `volumes` | `[]VolumeYAML` | Persistent named volumes (`name` + `path`) |
| `aliases` | `[]AliasYAML` | Host command aliases (`name` + `command`) |
| `security` | `SecurityConfig` | Container security: `privileged`, `cap_add`, `devices`, `security_opt`, `shm_size`, `group_add`, `mounts` |
| `port_relay` | `[]int` | Ports needing eth0->loopback socat relay (auto-adds socat dependency) |
| `secrets` | `[]SecretYAML` | Container secrets provisioned as Podman secrets at runtime (`name`, `target`, `env`) |
| `hooks` | `HooksConfig` | Lifecycle hooks: `post_enable` (runs after `ov config`), `pre_remove` (runs before `ov remove`) |
| `libvirt` | `[]string` | Raw libvirt XML snippets injected into VM domain XML after creation |
| `data` | `[]DataYAML` | Data mappings from layer directory to volume staging area (`src`, `volume`, `dest`) |
| `env_provides` | `map[string]string` | Env vars injected into OTHER containers when this service is deployed. Template: `{{.ContainerName}}` |
| `env_requires` | `[]EnvDependency` | Env vars this layer MUST have from the environment |
| `env_accepts` | `[]EnvDependency` | Env vars this layer CAN optionally use |
| `mcp_provides` | `[]MCPServerYAML` | MCP servers provided to OTHER containers when this service is deployed |
| `mcp_requires` | `[]EnvDependency` | MCP servers this layer MUST have from the environment |
| `mcp_accepts` | `[]EnvDependency` | MCP servers this layer CAN optionally use |

### port_relay

Ports needing an eth0->loopback socat relay inside the container. For services that bind only to 127.0.0.1 (like Chrome DevTools). Auto-adds socat dependency and generates a relay service via the configured init system (defined in init.yml).

Note: The chrome layer uses a dedicated `cdp-proxy` supervisord service instead of port_relay, to handle Chrome 146+ Host header validation for cross-container CDP.

```yaml
port_relay:
  - 9222
```

### secrets

Container secrets provisioned as Podman secrets at `ov config`/`ov start` time. Metadata only is stored in OCI image labels — never the secret value itself. Values are resolved from the credential store (keyring or config) at deployment time.

```yaml
secrets:
  - name: api-key                              # unique secret name
    target: /run/secrets/api_key               # mount path (default: /run/secrets/<name>)
    env: API_KEY                               # fallback env var (for Docker, or if Podman secrets fail)
```

- `name`: unique identifier, becomes Podman secret `ov-<image>-<name>`
- `target`: container mount path, defaults to `/run/secrets/<name>` if omitted
- `env`: optional fallback — if Podman secrets are unavailable (Docker), the value is injected as this env var instead

In quadlet mode, generates `Secret=ov-<image>-<name>,target=/run/secrets/<name>` directives. In direct mode, generates `--secret` flags.

### data

Data layers map files from the layer directory into volume staging areas. At build time, data files are COPYed into `/data/<volume>/[dest/]` in the image. At deploy time, `ov config` provisions them into bind-backed volumes; `ov update` merges new files non-destructively.

```yaml
data:
  - src: data/notebooks      # source dir relative to layer dir
    volume: workspace         # must match a volumes[].name in the image chain
    dest: ""                  # optional subdirectory within the volume
```

- `src`: path to a directory in the layer dir (e.g., `data/notebooks/`)
- `volume`: target volume name — must match a `volumes[].name` declared somewhere in the image's layer chain
- `dest`: optional subdirectory within the volume path (e.g., `examples/` would stage to `/data/<volume>/examples/`)

**Build behavior:** Data files are COPYed from the layer's scratch stage into `/data/<volume>/[dest/]` in the final image. This staging area is separate from volume mount points (which get overlaid at runtime).

**Deploy behavior:**
- `ov config --bind <volume>`: initial seed — copies staged data into empty bind mount directories
- `ov config --force-seed`: re-copies even if directory is not empty
- `ov update`: merges new files non-destructively (adds missing files, preserves existing)
- `ov update --force-seed`: overwrites matching files

**Data layers** are layers that only have `data:` and `volumes:` — no packages, no services, no install files. They're valid as standalone layers. Example: `/ov-layers:notebook-templates`.

**Data images** (`data_image: true` in images.yml) are scratch-based images containing only data layers — no OS, no runtime. Used as portable data bundles via `ov config --data-from <data-image>`.

### env_provides

Environment variables provided to OTHER containers when this service is deployed. Used for cross-container service discovery. Resolved at `ov config` time and stored in the global `env:` section of `deploy.yml`.

```yaml
env_provides:
  OLLAMA_HOST: "http://{{.ContainerName}}:11434"
  PGHOST: "{{.ContainerName}}"
  PGPORT: "5432"
```

- Template variable `{{.ContainerName}}` resolves to the container name (e.g., `ov-ollama`, or `ov-ollama-staging` with `--instance`)
- `env_provides` keys may overlap with `env` keys in the same layer — `env` is baked into the service's own image, `env_provides` is injected into consumers
- On `ov config remove` / `ov remove`, injected vars are automatically cleaned from deploy.yml
- `--update-all` flag on `ov config` propagates changes to all deployed quadlets immediately
- Validated by `ov validate`: empty keys and unknown template variables are errors
- Stored in OCI label `org.overthinkos.env_provides` for deploy-only scenarios

See `/ov:config` for `--update-all` flag and `/ov:deploy` for global env in deploy.yml.

### env_requires

Environment variables this layer MUST have from the environment to function. Enforced at `ov config` time with a **hard error** — missing required vars abort deployment. This is the enforcement contract (distinct from `env_accepts`, which is an opt-in allowlist).

```yaml
env_requires:
  - name: OLLAMA_API_KEY
    description: "API key for Ollama Cloud — required to authenticate outbound inference calls"
  - name: DATABASE_URL
    description: "Postgres connection URL"
    default: "postgres://localhost:5432/app"    # optional default; absent = hard fail
```

Each `EnvDependency` entry has `name` (required), `description` (required, shown in error output), `default` (optional — if set, missing var is substituted; if absent, missing var aborts `ov config`).

**Enforcement point:** `ov config` calls `checkMissingEnvRequires()` before writing the quadlet. A missing required var without a default is a fatal error — deploy.yml is not modified. The error message lists the missing vars with their descriptions so the user knows what to set with `-e`.

**Interaction with env_provides filtering:** When a provider service declares `env_provides` and the consumer declares matching `env_requires`, the provide-resolution pipeline (`provides.go`) **automatically satisfies** the requirement without the user needing `-e`. This is how cross-container service discovery works: deploy the provider first, then the consumer, and the required vars flow through deploy.yml's `provides:` section.

### env_accepts

Environment variables this layer CAN optionally use. **Opt-in allowlist** — no warnings if missing, but `env_accepts` also gates which `env_provides` vars actually get injected from other services. A provider's `env_provides` entry is **only** injected into consumers that declared matching `env_accepts` or `env_requires`. This prevents env var leakage: e.g., a tailscale sidecar's `TS_*` vars only reach containers that explicitly opt in.

```yaml
env_accepts:
  - name: HTTP_PROXY
    description: "Upstream HTTP proxy (optional)"
  - name: HTTPS_PROXY
    description: "Upstream HTTPS proxy (optional)"
  - name: NO_PROXY
    description: "Proxy exclusions — semicolons auto-converted to commas, container hostnames auto-enriched"
```

Same `EnvDependency` struct as `env_requires`. Used by the chrome layer for proxy accepts, and by most layers to opt into cross-container service discovery (e.g., `OLLAMA_HOST` from an ollama provider).

**Filtering model:** `env_provides` declarations are the supply side; `env_accepts` + `env_requires` are the demand side. The provide-resolution pipeline intersects the two sets per consumer, so env vars only reach containers that explicitly opt in. Missing `env_accepts` silently drops the var (opposite of missing `env_requires`, which is a hard fail). See `/ov:config` (Provides Filtering) and `/ov:sidecar` (Environment Contract) for the full lifecycle.

### mcp_provides

MCP servers provided to OTHER containers when this service is deployed. Used for cross-container MCP server discovery. Resolved at `ov config` time and stored in `deploy.yml` under `provides.mcp:`.

```yaml
mcp_provides:
  - name: jupyter
    url: "http://{{.ContainerName}}:8888/mcp"
    transport: http
```

- Each entry has `name` (required, unique), `url` (required, supports `{{.ContainerName}}` template), `transport` (optional, defaults to `"http"` for streamable HTTP; also supports `"sse"`)
- Consumer containers receive `OV_MCP_SERVERS` JSON env var with all resolved MCP server entries
- **Pod-aware**: When consumer and provider are in the same container, URLs resolve to `localhost` instead of container hostname. If both a local and remote entry share a name, local wins
- **No self-exclusion** (unlike env_provides): MCP servers are always included for same-container consumers
- On `ov config remove` / `ov remove`, MCP entries are automatically cleaned from deploy.yml
- Validated by `ov validate`: empty names, duplicate names, empty URLs, unknown template variables, and invalid transport values are errors
- Stored in OCI label `org.overthinkos.mcp_provides`

**Naming convention**: The `name:` field is the **service contract** — it should stay stable across layer/package/image renames. Pick a semantic name (like `jupyter`, `chrome-devtools`) that describes the service, not the artifact. See `/ov-layers:jupyter-mcp` § "MCP Name Decoupling" for a worked example: the `jupyter-colab-ml` layer, the `jupyter_colab_mcp` Python package, and the `jupyter-colab-ml-notebook` image were all renamed in one pass, but `mcp_provides.name: jupyter` stayed unchanged — and every consumer (`hermes`, `openwebui`, the `ov-jupyter` Claude Code plugin) kept working with zero edits.

See `/ov:config` for `--update-all` flag and `/ov:deploy` for provides section in deploy.yml.

### mcp_requires

MCP servers this layer MUST have from the environment to function. At `ov config` time, `warnMissingMCPRequires()` prints warnings for missing required servers.

```yaml
mcp_requires:
  - name: jupyter
    description: "JupyterLab CRDT MCP server for notebook manipulation"
```

Same `EnvDependency` struct as `env_requires` (`name`, `description`, optional `default`).

### mcp_accepts

MCP servers this layer CAN optionally use. No warnings if missing — purely for documentation and discoverability.

```yaml
mcp_accepts:
  - name: jupyter
    description: "JupyterLab CRDT MCP server for notebook manipulation"
```

Same struct as `mcp_requires`.

### Port Protocol Annotations

Ports support a protocol prefix that controls the backend scheme for tunnel commands, EXPOSE format, and port mapping.

```yaml
ports:
  - 18789                   # http (default) -> tailscale serve --https, cloudflared http://
  - "https+insecure:3000"   # https+insecure -> tailscale serve --https with https+insecure:// target
  - tcp:5900                # tcp -> tailscale serve --tcp, cloudflared tcp://
  - "tls-terminated-tcp:22" # tls-terminated-tcp -> tailscale serve --tls-terminated-tcp
  - udp:47998               # udp -> EXPOSE 47998/udp, not tunneled (warning)
  - 9222                    # http (default)
```

**Tailscale schemes:** `http` (default), `https`, `https+insecure`, `tcp`, `tls-terminated-tcp`
**Cloudflare schemes:** `http` (default), `https`, `tcp`, `ssh`, `rdp`, `smb`

UDP ports generate `EXPOSE <port>/udp` and are never tunneled. `ov validate` checks that port schemes are supported by the configured tunnel provider.

Ports with HTTPS backends (like Traefik with self-signed certs) MUST use `https+insecure` — plain `http` proxying to an HTTPS backend returns 404.

Protocol stored in OCI label `org.overthinkos.port_protos` (JSON map, only non-http entries). See `/ov:deploy` for tunnel backend scheme details.

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

### Pac (`pac:`)

Pacman packages for Arch Linux images. Activated when the image has `pac` in its `build` list.

```yaml
pac:
  packages:
    - neovim
    - ripgrep
  repos:
    - name: custom-repo
      server: https://example.com/repo/$arch
      siglevel: Optional TrustAll    # default if omitted
  options:
    - --needed
```

### AUR (`aur:`)

AUR packages installed via yay in a multi-stage build. The image must have `aur` in its `build` list and `builders.aur` configured (typically pointing to `archlinux-builder`). The builder image compiles the AUR packages, and the resulting `.pkg.tar.zst` files are copied to the final image and installed via `pacman -U`.

```yaml
aur:
  packages:
    - yay-bin
    - neovim-nightly-bin
  options:
    - --nocheck
```

**Status:** Working. AUR packages build successfully in the `archlinux-builder` multi-stage build. Debug packages are automatically disabled (makepkg.conf patched). Passwordless sudo is configured for the build user in the AUR build stage.

**Known limitation:** Post-build layer merging (`ov merge`) may fail with `payload does not match any of the supported image formats` on Arch-based images. The image builds and runs correctly — only the optional merge step fails.

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

Requires the init system's dependency layer (e.g., `supervisord` for containers). See init.yml `depends_layer`.

## Volume Declaration

```yaml
volumes:
  - name: data
    path: "~/.myapp"
```

Names must match `^[a-z0-9]+(-[a-z0-9]+)*$`. Docker/podman volume names become `ov-<image>-<name>`.

`CollectImageVolumes()` traverses the full image base chain (image -> base -> base's base), collecting volumes from all layers. Deduplicated by name (first declaration wins -- outermost image takes priority). Volumes are automatically mounted by `ov shell`, `ov start`, and `ov config`.

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
  group_add:
    - keep-groups
  mounts:
    - /dev/input:/dev/input:rw
    - tmpfs:/run/udev:rw,size=1m
  shm_size: "1g"
  memory_max: "8g"           # hard limit, OOM-kill above
  memory_high: "6g"          # soft limit, reclaim above
  memory_swap_max: "0"       # disable swap (recommended for containers)
  cpus: "4.0"                # CPU quota
```

Security settings are merged across layers: if any layer sets `privileged: true`, the result is privileged. `cap_add`, `devices`, `security_opt`, `group_add`, and `mounts` are unioned (deduplicated). Image-level `security:` in `images.yml` overrides `privileged` and appends to the other fields.

**`mounts`**: Host bind mounts or tmpfs mounts needed for device access. Format: `host:container:options` for bind mounts, `tmpfs:path:options` for tmpfs. Stored in the `org.overthinkos.security` image label and applied by `ov config`/`ov start`. Bind mounts generate `Volume=` in quadlets; tmpfs mounts generate `Tmpfs=`.

**Resource caps** (`memory_max`, `memory_high`, `memory_swap_max`, `cpus`): Cgroup limits applied at container runtime. **Smallest-wins merge** across layers — the most restrictive value from any layer wins. Image-level and deploy.yml values **replace** layer values (not merge). Emitted to quadlet as `MemoryMax=`, `MemoryHigh=`, `MemorySwapMax=`, `CPUQuota=` in the `[Service]` section; emitted to direct podman runs as `--memory`, `--memory-reservation`, `--memory-swap`, `--cpus`. Used by the chrome layer's crash-loop circuit breaker pattern: a tight `memory_max` triggers cgroup OOM-kill which supervisord's chrome-crash-listener catches and escalates to PID 1 termination, rebuilding the cgroup fresh. See `/ov-layers:chrome` (Resource Caps & Circuit Breaker) and `/ov-layers:supervisord` (Event Listeners) for the full pattern.

Source: `ov/security.go` (`CollectSecurity`, `SecurityArgs`), `ov/quadlet.go`, `ov/start.go`.

## Cache Mounts

| Step type | Cache path | Options |
|-----------|------------|---------|
| `rpm.packages`, `root.yml` (rpm) | `/var/cache/libdnf5` | `sharing=locked` |
| `deb.packages`, `root.yml` (deb) | `/var/cache/apt` + `/var/lib/apt` | `sharing=locked` |
| `pac.packages`, `aur`, `root.yml` (pac) | `/var/cache/pacman/pkg` | `sharing=locked` |
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

**Tier 1 in-tree Python packages**: If a layer ships its own Python package (e.g. `jupyter-mcp` ships the `jupyter_mcp` MCP server), the repository convention is a **nested directory layout** `layers/<layer-name>/<pkg-name>/<pkg-name>/`:

```
layers/jupyter-mcp/
  layer.yml
  user.yml               # runs `pip install --no-deps /ctx/jupyter_mcp`
  jupyter_mcp/           # distribution root
    pyproject.toml       # [project] name = "jupyter-mcp"; packages = ["jupyter_mcp"]
    jupyter_mcp/         # importable Python package
      __init__.py
      app.py
```

Two rules make this work cleanly across renames:
1. **Distribution name** (`pyproject.toml` → `[project].name`) and **Python package name** (inner directory) can be hyphenated (`jupyter-mcp`) or underscored (`jupyter_mcp`) — pick one convention and use it consistently.
2. **Internal imports must be relative** (`from .app import MCPExtensionApp`, never `from jupyter_mcp.app import MCPExtensionApp`). This means the package directory can be renamed without editing every `.py` file inside it — only `pyproject.toml`, the module docstrings, and external callers (`pip install` command, extension config files) need updating.

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

Add a `service` field to layer.yml with an init system program fragment (e.g., supervisord INI format). Add the init system's dependency layer to `depends` (e.g., `supervisord`).

## Style Guide

- Lowercase-hyphenated names for layers
- `root.yml` / `user.yml`: `all:` task for common logic, optional tag-specific tasks (`rpm:`, `pac:`, `fedora:`, etc.), idempotent
- System packages in `layer.yml` `rpm:`/`deb:`/`pac:` sections, not in root.yml
- AUR packages in `layer.yml` `aur:` section (requires `builders.aur` on the image)
- Python in `pixi.toml`, npm in `package.json`, Rust in `Cargo.toml`
- Binary downloads in `root.yml`: detect arch with `uname -m`, map via `case`
- Never `pip install`, `conda install`, or `dnf install python3-*`
- Never `dnf clean all` or `pacman -Scc` in root.yml (cache mounts handle it)

## Cross-References

- `/ov:image` -- Adding layers to image definitions
- `/ov:build` -- Building images with layers
- `/ov:generate` -- Containerfile generation and layer scratch-stage cache behavior (`ov build --no-cache` caveat lives in `/ov:build`)
- `/ov:config` -- `env_requires`/`env_accepts` enforcement, provides filtering, resource caps, `--update-all`
- `/ov:deploy` -- Provides configuration in deploy.yml (env + MCP), resource caps, tunnel is deploy.yml-only
- `/ov:sidecar` -- Sidecars as env_provides participants (tailscale `TS_*` filtering)
- `/ov-layers:chrome` -- Canonical consumer of `env_accepts` (proxy vars) and resource caps (crash-loop circuit breaker)
- `/ov-layers:supervisord` -- Event listener pattern triggered by resource caps

## When to Use This Skill

**MUST be invoked** when the task involves layer authoring, layer.yml, root.yml, user.yml, pixi.toml, package.json, Cargo.toml, or any file under layers/. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Pre-build. Author layers before adding to images. See also `/ov:image` (image composition), `/ov:build` (building).
