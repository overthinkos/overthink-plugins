---
name: layer
description: |
  MUST be invoked before any work involving: layer authoring, layer.yml, tasks, pixi.toml, package.json, Cargo.toml, or any file under layers/. This skill is the authoritative reference for the `tasks:` verb catalog, `vars:` substitution, execution order, and per-verb validation. Every other skill defers here for install-schema questions.
---

# Layer - Layer Authoring

## Overview

A **layer** is a directory under `layers/<name>/` that installs a single concern. Layers are the building blocks of container images in overthink. Each layer declares its packages, environment variables, services, volumes, and **install tasks** in a single `layer.yml` file.

Since the `tasks:` refactor, there is **one YAML file per layer** for install logic — no separate Taskfiles, no `tasks:` / `tasks:`. Everything an author needs to install flows through `tasks:` and auto-detected package manifests (`pixi.toml`, `package.json`, `Cargo.toml`).

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Scaffold new layer | `ov image new layer <name>` | Create layer directory with starter `layer.yml` (see `/ov:new`) |
| Edit a layer field | `ov layer set <name> <dotpath> <value>` | Comment-preserving YAML edit by dot-path |
| Append rpm/deb/pac/aur packages | `ov layer add-rpm <name> <pkg…>` (plus `add-deb`, `add-pac`, `add-aur`) | Idempotent append; auto-upgrades scaffold's null `packages:` to a sequence |
| Write a free-form file (`pixi.toml`, `root.yml`, …) | `ov image write <rel-path> --content X` | Escape hatch for files the schema setters don't cover; guarded against `..` traversal |
| List all layers | `ov image list layers` | Show available layers from filesystem |
| List services | `ov image list services` | Layers with `service` in layer.yml |
| List volumes | `ov image list volumes` | Layers with `volumes` in layer.yml |
| List aliases | `ov image list aliases` | Layers with `aliases` in layer.yml |
| Validate | `ov image validate` | Check all layers and images |

Every editor verb above auto-becomes an MCP tool via Kong reflection (`layer.set`, `layer.add-rpm`, `image.write`, …) so an agent driving `ov mcp serve` can author layers from scratch over RPC without touching the filesystem directly. See `/ov:mcp` "Authoring tools" for the worked end-to-end example, and `/ov:new` for the project / image / layer scaffolders that bootstrap the flow.

### Editing layer.yml via the CLI (no hand-edit required)

The `ov layer …` group edits `layers/<name>/layer.yml` through the `yaml.v3` Node API, so **comments and key order are preserved** across edits. Unlike unmarshal-then-marshal, nothing gets scrambled when an agent (or a shell script) touches the file:

```bash
# Append packages (idempotent; handles scaffold's null `packages:` value):
ov layer add-rpm sshd openssh-server openssh-clients
ov layer add-deb sshd openssh-server
ov layer add-pac sshd openssh

# Set any field by dot-path (value is parsed as YAML):
ov layer set sshd env.SSHD_PORT 22
ov layer set sshd service.name sshd
ov layer set sshd ports '["22:22"]'
ov layer set sshd depends '[supervisord]'

# Free-form files (layer scripts, pixi.toml, root.yml, *.service):
ov image write layers/sshd/root.yml --content 'tasks:\n  - cmd: echo configured\n'
ov image cat layers/sshd/root.yml
```

Implementation: `ov/scaffold_cmds.go` (verbs) + `ov/yaml_setter.go` (`SetByDotPath`). Tested in `ov/yaml_setter_test.go` — the comment-preservation guarantee is explicitly exercised (leading file comments, sibling keys, and per-key inline comments all survive round trips). See `/ov-dev:go` "Implementation insights" for the full rationale.

## Install Surface (what a layer directory holds)

A layer directory can contain any combination of these:

| Artifact | Runs as | Purpose |
|---|---|---|
| `layer.yml` `rpm:`/`deb:`/`pac:`/`aur:` sections | root | System packages declared declaratively |
| `layer.yml` `tasks:` list | per-task `user:` | Ordered install operations — the primary extension point (see catalog below) |
| `pixi.toml` / `pyproject.toml` / `environment.yml` | user (builder stage) | Python/conda packages. Multi-stage build. Only one per layer |
| `package.json` | user (builder stage) | npm packages — installed globally via `npm install -g` |
| `Cargo.toml` + `src/` | user (builder stage) | Rust crate — built via `cargo install --path` |
| `build.sh` | user (builder stage) | Optional post-install script for pixi layers. Runs in the pixi builder after `pixi install`. For build-time logic that can't be expressed in pixi.toml (C extension compilation, npm builds, binary patching). |

**Auto-detection:** The build system scans each layer directory for these files. `pixi.toml`, `pyproject.toml`, `environment.yml`, `package.json`, and `Cargo.toml` trigger automatic multi-stage builds — no manual install commands needed. Use `tasks:` only for things those manifests can't express (binary downloads, file copies from `/ctx`, inline config writes, post-install configuration, `go install`, etc.).

**Root vs user rule:** Pixi/npm/cargo builders always run as user — never as root. For `tasks:`, the `user:` field per task is explicit: `user: root` for system-wide changes, `user: ${USER}` for anything under `${HOME}`, or a literal username for custom users (must be created earlier in the same layer via a `cmd:` task).

---

## Task Verb Catalog

Every task in `tasks:` is a YAML map with **exactly one verb key** (the discriminator) plus optional sibling modifiers. The verb's value is the primary argument. `ov image validate` rejects tasks with zero verbs or multiple verbs.

| Verb | Value | Required modifiers | Optional modifiers | Purpose |
|---|---|---|---|---|
| `cmd:` | shell command (multi-line OK) | — | `user`, `comment` | Arbitrary shell — last-resort escape hatch |
| `mkdir:` | directory path | — | `user`, `mode`, `comment` | Create directory (`mkdir -p`; coalesces with adjacent) |
| `copy:` | source relative to layer dir | `to:` | `user`, `mode`, `comment` | `COPY --from=<layer-stage> --chmod= [--chown=]` — no RUN |
| `write:` | destination path | `content:` | `user`, `mode`, `comment` | Write inline content — staged + COPY, no shell heredoc |
| `link:` | symlink path (where the link lives) | `target:` | `user`, `comment` | `ln -sf <target> <link>` (coalesces with adjacent) |
| `download:` | URL | — (`to:` unless `extract: sh`) | `user`, `extract`, `to`, `include`, `strip_components`, `mode`, `env`, `comment` | `curl` + optional extract (`tar.gz`/`tar.xz`/`tar.zst`/`zip`/`none`/`sh`). `strip_components: N` (2026-04) emits `tar --strip-components=N` for tar.* — drops leading path segments so tarballs that nest under a top-level arch/version dir (Go, Rust, Node binary releases) land files directly at `to:`. See `/ov-layers:uv` for the canonical uv-x86_64-unknown-linux-gnu/uv → /usr/local/bin/uv example. |
| `setcap:` | file path | — | `user` (implicit root), `caps`, `comment` | File capabilities (`setcap -r` strip if `caps` empty) |
| `build:` | `"all"` | — | `user` (default `${USER}`), `comment` | Run auto-detected builders (pixi/npm/cargo/aur) at this point (instead of end-of-layer) |

### Shared modifiers

| Modifier | Applies to | Default | Purpose |
|---|---|---|---|
| `user` | all verbs | `root` | `root` / `${USER}` / literal username / `<uid>:<gid>` |
| `mode` | `mkdir`, `copy`, `write`, `download` | type-specific (`0755`/`0644`) | Octal permissions |
| `to` | `copy`, `download` | — | Destination in container |
| `target` | `link` | — | What the symlink points to |
| `content` | `write` | — | Inline file body (YAML block scalar) |
| `extract` | `download` | `none` | Archive format — `tar.gz` / `tar.xz` / `tar.zst` / `zip` / `none` / `sh` |
| `include` | `download` | — | Extract only these paths (archive formats) |
| `env` | `download` | — | Env vars for install scripts (`sh` extract) |
| `caps` | `setcap` | empty (= strip) | Capability spec (e.g. `cap_setuid=ep`) |
| `comment` | all | — | Emitted as a Containerfile comment above the task |

### Verb examples

```yaml
# Arbitrary shell
- cmd: dnf install -y https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm
  user: root

# Multi-line shell (shared shell — cd persists)
- cmd: |
    git clone --depth 1 https://github.com/foo/bar /tmp/src
    cd /tmp/src
    cargo build --release
    install -m 755 target/release/bar /usr/local/bin/bar
    rm -rf /tmp/src
  user: root

# Make directory
- mkdir: /etc/traefik/dynamic
  user: root
  mode: "0755"

- mkdir: "${HOME}/.config/sway"
  user: "${USER}"

# Copy existing file from layer dir to container (no RUN, pure COPY)
- copy: traefik.yml
  to: /etc/traefik/traefik.yml
  mode: "0644"
  user: root

# Write new file from inline YAML content (no shell heredoc!)
- write: /etc/X11/xorg.conf
  mode: "0644"
  user: root
  content: |
    Section "ServerFlags"
      Option "DontVTSwitch" "true"
    EndSection

# Create symlink
- link: /usr/local/bin/node
  target: /usr/bin/node-24
  user: root

# Download + extract
- download: "https://github.com/traefik/traefik/releases/download/${TRAEFIK_VERSION}/traefik_${TRAEFIK_VERSION}_linux_${ARCH}.tar.gz"
  extract: tar.gz
  to: /usr/local/bin
  include: [traefik]
  user: root

# Install script piped into shell
- download: "https://astral.sh/uv/install.sh"
  extract: sh
  env:
    UV_INSTALL_DIR: /usr/local/bin
  user: root

# File capabilities
- setcap: /usr/bin/sway        # no caps → strip (setcap -r)

- setcap: /usr/bin/newuidmap
  caps: "cap_setuid=ep"

# Explicit builder placement (lets you run tasks AFTER pixi/npm/cargo)
- build: all
  user: "${USER}"
- cmd: pip install --no-deps ./vllm-nightly.whl
  user: "${USER}"
```

### `copy:` vs `write:` — never conflate

They happen to emit `COPY` directives under the hood, but they have entirely distinct semantics:

| | `copy` | `write` |
|---|---|---|
| Source of bytes | File on disk under `layers/<name>/` | Inline `content:` block in `layer.yml` |
| Replaces old pattern | `cp /ctx/foo bar` + `chmod` | `cat > foo << 'EOF' … EOF` |
| Validation check | `src` must exist under layer dir at generate time | `content` must be non-empty |
| Cache key | Layer-stage file content | Content-addressed staged file at `.build/<image>/_inline/<layer>/<sha256>` |
| Shell-heredoc involvement | None | **None** — content never appears in the Containerfile |

`write:` is the heredoc-safe path: the YAML `content:` is staged to disk at generate time and delivered via `COPY`. That means `$`, backticks, nested `EOF` markers, and arbitrary binary-safe text are all handled without any escaping. If you find yourself writing `cat > foo << 'EOF'` inside a `cmd:` block, use `write:` instead.

---

## `vars:` and `${VAR}` Substitution

`vars:` is a layer-local `map[string]string`. Values are emitted as `ENV` before the layer's tasks, so every subsequent directive in the layer (including `COPY --chmod=` paths) sees them as shell-resolvable `${VAR}` references.

```yaml
vars:
  TRAEFIK_VERSION: v3.4.0
  NIRI_BRANCH: feat/virtual

tasks:
  - download: "https://github.com/traefik/traefik/releases/download/${TRAEFIK_VERSION}/traefik_${TRAEFIK_VERSION}_linux_${ARCH}.tar.gz"
    extract: tar.gz
    to: /usr/local/bin
    include: [traefik]
    user: root
```

### Auto-exports

These names are reserved — `vars:` may not shadow them:

| Name | Value | Resolution |
|---|---|---|
| `USER` | image's configured username | Generate-time (from image.yml resolution) |
| `UID` | numeric user ID | Generate-time |
| `GID` | numeric group ID | Generate-time |
| `HOME` | resolved home directory | Generate-time |
| `ARCH` | BuildKit-style: `amd64` / `arm64` / `ppc64le` / `s390x` | Build-time via `ARG TARGETARCH` + `ENV ARCH=${TARGETARCH}` at layer top |
| `BUILD_ARCH` | uname-style: `x86_64` / `aarch64` / … | Build-time, shell-only (auto-injected inside `cmd:` and `download:` as `BUILD_ARCH=$(uname -m)`) |

**Why two `ARCH` flavours:** `${ARCH}` is BuildKit form because that's what most upstream release naming uses (traefik, mcp-grafana, cosign, etc.). `${BUILD_ARCH}` is uname form because a handful of projects (pixi, yay, just, typst) use `x86_64`/`aarch64` in their release filenames. Pick whichever matches your URL template — the generator handles both.

### Where substitution applies

`${VAR}` is resolved in every task field **except**:

- `cmd:` command text — passed verbatim to bash so shell-style `${VAR:-default}`, `$(command)`, and `$NAME` all work the way you'd expect
- `write: content:` — the file body is verbatim bytes (never substituted, never heredoc'd)

Everything else (paths, URLs, modes, `to`, `target`, `user`, etc.) resolves via the Docker-level ENV mechanism, so BuildKit sees the values and substitutes in COPY/RUN/ENV directives.

### Validation

- `vars:` keys must match `^[A-Z_][A-Z0-9_]*$` (standard shell identifier)
- Keys may not collide with auto-exports or with the layer's own `env:` keys
- Unresolved `${VAR}` in non-shell fields (paths, URLs, etc.) errors at `ov image validate`

---

## Style: Explicit Tasks, No YAML Anchors

Every task — root or user — must be written out in full. Do **not** use YAML anchors (`&name`, `<<: *name`) to share `user:` / `mode:` across tasks. `gopkg.in/yaml.v3` parses anchors correctly, but the repo convention is that every `layer.yml` task reads end-to-end without indirection.

```yaml
# Style across the repo — root and user tasks look identical in shape:
tasks:
  - mkdir: /etc/traefik
    user: root
    mode: "0755"

  - mkdir: "${HOME}/.local/bin"
    user: "${USER}"

  - copy: chrome-wrapper
    to: "${HOME}/.local/bin/chrome-wrapper"
    mode: "0755"
    user: "${USER}"
  - copy: chrome-restart
    to: "${HOME}/.local/bin/chrome-restart"
    mode: "0755"
    user: "${USER}"
```

Why: anchor merges always override the verb (the interesting field), so the DRY saving is 1–2 trivial fields per task at the cost of forcing readers to jump to the anchor definition to understand each task. Adjacent-coalescing in the generator already collapses identical consecutive operations into one Containerfile directive — there is no build-time cost to explicit repetition.

---

## Execution Order

**Tasks run in the exact YAML order you wrote them. No reordering, ever.** This is load-bearing: `download` before `copy` that references the downloaded path; `mkdir` before `copy` into that directory; `cmd useradd` before `cmd` as that new user.

### Adjacent-coalescing (the only rendering optimisation)

When two or more consecutive tasks share the same verb AND the same resolved `user:` AND the verb supports batching, they render into one Containerfile directive instead of N. This is pure output optimisation — the operations still happen in authored sequence, and no task ever moves past a different-verb or different-user task.

| Verb | Coalesces adjacent same-user? | Output |
|---|---|---|
| `mkdir` | Yes | `RUN mkdir -p p1 p2 p3 …` (one RUN; `chmod` grouped by mode) |
| `link` | Yes | `RUN ln -sf t1 l1 && ln -sf t2 l2 …` |
| `setcap` | Yes | `RUN setcap …` chain |
| `copy` | No coalescing needed | Multiple `COPY` directives back-to-back, no RUN between them |
| `write` | No coalescing needed | Same as `copy` but from staged-inline content |
| `download` | No | Each URL gets its own RUN — merging would hide failures |
| `cmd` | No | Each `cmd:` maps 1:1 to one RUN — merging arbitrary shell would erase author intent |
| `build` | Singleton | One per layer (auto or explicit) |

**Non-adjacent same-verb tasks never coalesce.** `mkdir /a` → `copy foo /a/foo` → `mkdir /b` renders as three directives, not two. Collapsing the two mkdirs would change observable state (the copy would see a different directory layout) and is forbidden.

### `mkdir -p` ancestor registration

When you declare `mkdir: /a/b/c`, the generator also treats `/a` and `/a/b` as "declared" — because Unix `mkdir -p` creates the full path. That means a later `copy` into `/a/b/foo` won't trigger a redundant auto-inserted `mkdir /a/b`. This matches author intent: you wrote one mkdir, you get one mkdir, even if multiple COPYs land under the same tree.

### Parent-directory auto-insertion

The single, tightly-scoped exception to "no implicit inserts": if a `copy:` or `write:` task's destination parent isn't already declared (directly or via ancestor-registration above), the generator prepends a single `RUN mkdir -p <parent>` (as the task's `user:`) **immediately before** the COPY — never relocating anything, never merging across other tasks. To opt out of auto-insertion for a specific path, declare the parent explicitly with `mkdir:` earlier in the list.

### USER directives

The generator tracks the running USER and emits `USER <value>` only when the next task's resolved user differs. Eight consecutive `${USER}` tasks get one `USER 1000` directive followed by eight directives — no redundant switches. Between different-user tasks, a `USER` switch happens where author order requires.

---

## User Resolution

The `user:` field accepts:

| Value | Emits | COPY `--chown` pair |
|---|---|---|
| `root` / `"0"` / omitted | `USER 0` | — (root is the COPY default) |
| `${USER}` | `USER <numeric UID>` | `<UID>:<GID>` (e.g. `1000:1000`) |
| `<uid>:<gid>` (e.g. `1010:1010`) | `USER <uid>:<gid>` | same |
| bare numeric `<uid>` | `USER <uid>` | `<uid>:<uid>` |
| literal name (`postgres`, `immich`) | `USER <name>` | `<name>:<name>` |

**Why `${USER}` resolves to a numeric UID:** matches the pre-refactor `USER <UID>` convention and avoids a `/etc/passwd` dependency at the instant the switch happens. Base images create the named user, but numeric always works even if `/etc/passwd` hasn't been read yet by whatever shell the RUN spawns.

**User creation is explicit.** If you reference a literal username, an earlier `cmd:` task (as root) must `useradd` them. The generator does not auto-create users. Example:

```yaml
tasks:
  - cmd: |
      useradd -r -u 1010 -g root -s /sbin/nologin immich
      mkdir -p /srv/immich && chown -R immich:root /srv/immich
    user: root
  - cmd: |
      cd /srv/immich
      pnpm install --frozen-lockfile
      pnpm build
    user: immich
```

---

## layer.yml Field Reference

| Field | Type | Purpose |
|-------|------|---------|
| `version` | `string` | CalVer (`YYYY.DDD.HHMM`) of this layer definition. Set manually. |
| `status` | `string` | `working`, `testing`, or `broken`. Default: `testing`. |
| `info` | `string` | Free-form description of what works / doesn't. Recommended for `testing` / `broken`. |
| `depends` | `[]string` | Layer dependencies. Resolved transitively, topologically sorted. |
| `layers` | `[]string` | Compose other layers into this one (splicing). |
| `env` | `map[string]string` | Container-runtime environment variables. Merged across layers. |
| `path_append` | `[]string` | Paths appended to `$PATH`. Deduplicated. |
| `vars` | `map[string]string` | **Build-time** layer-local variables for `${VAR}` substitution. Emitted as `ENV` before tasks. |
| `tasks` | `[]Task` | **Ordered** install operations. See Task Verb Catalog above. |
| `ports` | `[]int \| []PortSpec` | Exposed ports (1-65535). Protocol-annotated form: `tcp:5900`, `https+insecure:3000`, etc. |
| `route` | `{host, port}` | Traefik reverse proxy route. |
| `service` | multiline string | Supervisord `[program:<name>]` fragment. |
| `rpm` / `deb` / `pac` / `aur` | object | Per-format package declarations (see Package Manager Sections). |
| `volumes` | `[]{name, path}` | Persistent named volumes. |
| `aliases` | `[]{name, command}` | Host command aliases. |
| `security` | object | Container security: `privileged`, `cap_add`, `devices`, `security_opt`, `shm_size`, resource caps. |
| `port_relay` | `[]int` | Ports needing eth0 → loopback socat relay. Auto-adds `socat` dependency. |
| `secrets` | `[]SecretYAML` | Image-owned container secrets (auto-generated per instance). |
| `hooks` | `HooksConfig` | Lifecycle hooks: `post_enable` (after `ov config`), `pre_remove` (before `ov remove`). |
| `libvirt` | `[]string` | Raw libvirt XML snippets for VM domain XML injection. |
| `data` | `[]DataYAML` | Data mappings (`src` → volume `dest`) for volume staging. |
| `env_provides` | `map[string]string` | Env vars injected into OTHER containers when this service is deployed. Template: `{{.ContainerName}}`. |
| `env_requires` | `[]EnvDependency` | Plaintext env vars this layer MUST have. Hard error at `ov config` if missing. |
| `env_accepts` | `[]EnvDependency` | Plaintext env vars this layer CAN optionally use. Opt-in allowlist for `env_provides` injection. |
| `secret_accepts` / `secret_requires` | `[]EnvDependency` | Credential-backed env vars. Values live in credential store, never in deploy.yml/quadlet. |
| `mcp_provides` / `mcp_requires` / `mcp_accepts` | various | MCP server discovery analogous to `env_*`. |
| `system_services` | `[]string` | System-level systemd units to enable (bootc images only). |

Field details for non-`tasks:` sections are below; they're unchanged from the pre-refactor schema.

---

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

AUR packages installed via yay in a multi-stage build. The image must have `aur` in its `build` list and `builders.aur` configured (typically `archlinux-builder`). The builder compiles AUR packages; resulting `.pkg.tar.zst` files are copied into the final image and installed via `pacman -U`.

```yaml
aur:
  packages:
    - yay-bin
    - neovim-nightly-bin
  options:
    - --nocheck
```

---

## Dependencies

Layers declare dependencies via `depends`. The generator resolves transitively, topologically sorts, and pulls missing dependencies automatically. Circular dependencies are a validation error.

```yaml
depends:
  - python
  - supervisord
```

### `depends` vs `layers`

| | `depends` | `layers` |
|---|---|---|
| Purpose | Prerequisite ordering | Group composition |
| Effect | Ensures dependency is installed first | Splices layers at this layer's position |
| Transitive | Yes — pulls in sub-dependencies | Yes — recursively expands |
| Typical use | Runtime/build prerequisites | Metalayers, layer bundles |

**Common mistake:** `depends: [pixi]` when you mean `depends: [python]`. The `pixi` layer installs the pixi binary (build tool). The `python` layer installs Python via pixi. Your layer needs Python.

---

## Environment Variables

```yaml
env:
  PIXI_CACHE_DIR: "~/.cache/pixi"
  MY_VAR: "value"

path_append:
  - "~/.pixi/bin"
  - "~/.local/bin"
```

`~` and `$HOME` expand to the resolved home directory at generation time. Setting `PATH` directly in `env` is a validation error — use `path_append`. Later layers override earlier for the same key.

**`env:` vs `vars:`:** `env:` is container **runtime** environment (emitted as `ENV` and persists into the running container). `vars:` is **build-time** substitution for `${VAR}` references inside `tasks:` — also emitted as `ENV` so BuildKit can substitute in COPY paths, but conceptually scoped to the layer's install. There's no hard rule against using `env:` for both purposes, but keeping them separate makes intent clearer.

**`env:` is a MAP, not a list.** The YAML parser decodes it as `map[string]string`, not `[]string`. Authoring it as `- KEY=value` fails with `cannot unmarshal !!seq into map[string]string` at `ov image validate`. Always use map form:

```yaml
# ❌ WRONG — parser rejects the list shape
env:
  - OV_PROJECT_DIR=/workspace
  - GOPATH=~/go

# ✓ RIGHT — map form
env:
  OV_PROJECT_DIR: "/workspace"
  GOPATH: "~/go"
```

The `ov-mcp` layer is the canonical example of the map form used to thread a container-level env var into the MCP server process via supervisord.

---

## Service Declaration

```yaml
service: |
  [program:myservice]
  command=/usr/bin/myservice --flag
  autostart=true
  autorestart=true
```

Requires the init system's dependency layer (e.g., `supervisord` for containers). See build.yml `init.<name>.depends_layer`.

## Volume Declaration

```yaml
volumes:
  - name: data
    path: "~/.myapp"
```

Names must match `^[a-z0-9]+(-[a-z0-9]+)*$`. Docker/podman volume names become `ov-<image>-<name>`. Collected across the full image base chain; first declaration wins.

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
  memory_max: "8g"           # hard limit
  memory_high: "6g"          # soft limit
  memory_swap_max: "0"       # disable swap
  cpus: "4.0"                # CPU quota
```

Security settings merge across layers (union for lists; `privileged` true if any layer sets it; smallest-wins for resource caps). Image-level `security:` in `image.yml` overrides `privileged` and replaces resource caps.

Resource caps (memory / cpus) are used by the chrome layer's crash-loop circuit breaker. See `/ov-layers:chrome` and `/ov-layers:supervisord` for the event-listener pattern.

---

## port_relay

Ports needing an eth0 → loopback socat relay inside the container. For services that bind only to 127.0.0.1. Auto-adds `socat` dependency and generates a relay service.

```yaml
port_relay:
  - 9222
```

Note: the chrome layer uses a dedicated `cdp-proxy` supervisord service instead of `port_relay`, to handle Chrome 146+ Host header validation.

## Port Protocol Annotations

Ports support a protocol prefix that controls tunnel backend scheme and EXPOSE format:

```yaml
ports:
  - 18789                   # http (default) → tailscale --https, cloudflared http://
  - "https+insecure:3000"   # https+insecure → HTTPS tunnel with self-signed OK
  - tcp:5900                # raw TCP
  - "tls-terminated-tcp:22" # tailscale TLS-terminated-tcp
  - udp:47998               # EXPOSE /udp, not tunneled
  - 9222                    # http (default)
```

Tailscale schemes: `http`, `https`, `https+insecure`, `tcp`, `tls-terminated-tcp`. Cloudflare adds `ssh`, `rdp`, `smb`. HTTPS backends (Traefik with self-signed certs) MUST use `https+insecure` — plain `http` proxying to HTTPS returns 404.

---

## secrets (image-owned)

```yaml
secrets:
  - name: api-key                # Podman secret `ov-<image>-<name>`
    target: /run/secrets/api_key # mount path (default: /run/secrets/<name>)
    env: API_KEY                 # fallback env var if Podman secrets unavailable
```

Metadata only lives in OCI labels. Values are auto-generated per instance at `ov config` time. Use for image-internal secrets (like `db-password`). For user-owned credentials (API keys, auth tokens), use `secret_accepts` / `secret_requires` instead.

---

## env_provides / env_requires / env_accepts

Cross-container environment-variable service discovery. `env_provides` is the supply side; `env_requires` (mandatory) and `env_accepts` (opt-in) are the demand side.

```yaml
env_provides:
  OLLAMA_HOST: "http://{{.ContainerName}}:11434"
  PGHOST: "{{.ContainerName}}"
  PGPORT: "5432"

env_requires:
  - name: DATABASE_URL
    description: "Postgres connection URL"
    default: "postgres://localhost:5432/app"

env_accepts:
  - name: HTTP_PROXY
    description: "Upstream HTTP proxy (optional)"
```

`{{.ContainerName}}` resolves at `ov config` time. `env_provides` values are injected only into consumers that declare matching `env_accepts` or `env_requires` (opt-in filtering — prevents env var leakage). Missing `env_requires` without a default is a hard error at `ov config`; missing `env_accepts` silently drops the var.

See `/ov:config` (`--update-all` flag, provides filtering) and `/ov:deploy` (deploy.yml `provides:` section) for the full lifecycle.

## secret_accepts / secret_requires

Credential-backed env vars. Same YAML shape as `env_accepts` / `env_requires`, but values flow through the credential store (keyring → kdbx → config) and arrive via Podman secrets — **never plaintext in deploy.yml or the quadlet**.

```yaml
secret_requires:
  - name: WEBUI_ADMIN_PASSWORD
    description: "Initial admin account password"

secret_accepts:
  - name: OPENROUTER_API_KEY
    description: "API key for OpenRouter LLM inference"
    key: ov/api-key/openrouter     # optional override; default ov/secret/<NAME>
```

Use for API keys, passwords, auth tokens. `key:` override must match `^ov/<service>/<key>$` (lowercase). Multiple layers sharing the same upstream credential (e.g. `ov/api-key/openrouter`) all resolve to the same stored value. See `/ov:secrets` for the credential-store chain, rotation, and `-e NAME=VAL` auto-import.

## mcp_provides / mcp_requires / mcp_accepts

Cross-container MCP server discovery. Consumers receive `OV_MCP_SERVERS` as a JSON env var at `ov config` time.

```yaml
mcp_provides:
  - name: jupyter
    url: "http://{{.ContainerName}}:8888/mcp"
    transport: http                # or "sse"

mcp_accepts:
  - name: jupyter
    description: "JupyterLab CRDT MCP server for notebook manipulation"
```

**Pod-aware:** when provider and consumer share a container, URLs resolve to `localhost` (local wins over remote for same-named entries). **Naming is the service contract** — keep `name:` stable across layer/package/image renames.

**Testing the endpoint:** once a layer is deployed, `ov test mcp ping <image>` verifies the server is alive, and `ov test mcp list-tools <image>` enumerates the tool catalog. Both are authorable as deploy-scope `mcp:` declarative checks inside the layer's `tests:` block. The full verb reference (methods, URL rewriting, port-publishing gotcha, validator rules) lives in `/ov:mcp`.

---

## data

Data layers stage files from the layer directory into volume bind-mount areas. Build-time: files COPY into `/data/<volume>/[dest/]`. Deploy-time: `ov config --bind <volume>` provisions them into bind directories; `ov update` merges non-destructively.

```yaml
volumes:
  - name: workspace
    path: "~/workspace"

data:
  - src: data/notebooks
    volume: workspace
    dest: ""                 # optional subdirectory within volume
```

**Data layers** are layers with only `data:` + `volumes:` — no packages, no services, no tasks. Valid standalone. **Data images** (`data_image: true` in image.yml) are scratch-based — consumed via `ov config --data-from <image>`. See `/ov-layers:notebook-templates` for a worked example.

---

## Cache Mounts (used by the generator)

The generator attaches cache mounts automatically based on task context:

| Emission site | Cache path | Options |
|---|---|---|
| `rpm.packages` (dnf install) | `/var/cache/libdnf5` | `sharing=locked` |
| `deb.packages` (apt install) | `/var/cache/apt` + `/var/lib/apt` | `sharing=locked` |
| `pac.packages` + `aur` | `/var/cache/pacman/pkg` | `sharing=locked` |
| `cmd:` as root | distro-format caches (as above) + `/ctx` bind to layer stage | — |
| `cmd:` as non-root | `/tmp/npm-cache` (uid-scoped) + `/ctx` bind | `uid=<UID>,gid=<GID>` |
| `download:` | `/tmp/downloads` (shared across layers) | — |
| pixi builder stage | `/tmp/pixi-cache` + `/tmp/rattler-cache` | `uid=<UID>,gid=<GID>` |
| npm builder stage | `/tmp/npm-cache` | `uid=<UID>,gid=<GID>` |
| cargo inline | `/tmp/cargo-cache` | `uid=<UID>,gid=<GID>` |

UID/GID in non-root cache mounts are dynamic (from resolved image config). Flat `/tmp/<tool>-cache` paths avoid buildah permission issues with nested paths.

---

## Common Workflows

### Create a new layer

```bash
ov image new layer my-tool
# Edit layers/my-tool/layer.yml — add packages, deps, env, tasks
ov image validate
```

### Add system packages

Add an `rpm:` / `deb:` / `pac:` / `aur:` section to `layer.yml`. Multi-distro layers declare all sections — the generator picks the one matching the image's `build:` list. For distro-version overrides, use a tag section like `fedora:43:` (checked first; first match wins).

### Add Python packages

Create `pixi.toml` in the layer directory. **`layer.yml` must depend on `python`, not `pixi`** (the `pixi` layer installs the pixi binary; the `python` layer installs Python via pixi). Never use `pip install`, `conda install`, `pixi global install`, or `uv tool install` inside a `cmd:` task. Always use `pixi.toml`.

**In-tree Python packages** shipped by a layer live at `layers/<layer-name>/<pkg-name>/<pkg-name>/` with `pyproject.toml` at the distribution root. Internal imports must be relative (`from .app import X`) so the package directory can be renamed without editing every `.py` file.

### Add npm packages

Create `package.json` in the layer directory; `layer.yml` depends on `nodejs`. The build system runs `npm install -g` in a multi-stage build. Do **not** put `npm install -g` inside a `cmd:` task — `package.json` is the declarative path.

### Add Go packages

Go has no declarative manifest for global installs, so use `cmd:`:

```yaml
depends:
  - golang

env:
  GOPATH: "~/go"

path_append:
  - "~/go/bin"

tasks:
  - cmd: |
      go install github.com/org/tool/cmd/tool@latest
      go clean -cache
    user: "${USER}"
```

For cgo dependencies, add the required `-devel` packages to `rpm:`. Always `go clean -cache` to shrink the image.

### Add a binary download

```yaml
vars:
  TOOL_VERSION: v1.2.3

tasks:
  - download: "https://github.com/org/tool/releases/download/${TOOL_VERSION}/tool-linux-${ARCH}.tar.gz"
    extract: tar.gz
    to: /usr/local/bin
    include: [tool]
    user: root
```

Use `${ARCH}` (BuildKit-style) if the release URL uses `amd64`/`arm64`; use `${BUILD_ARCH}` if it uses `x86_64`/`aarch64`. If the project only ships x86_64, omit the template and hardcode — don't fake multi-arch.

### Install a wrapper script + make it the default

```yaml
tasks:
  - mkdir: "${HOME}/.local/bin"
    user: "${USER}"
  - copy: my-wrapper
    to: "${HOME}/.local/bin/my-wrapper"
    mode: "0755"
    user: "${USER}"
  - link: "${HOME}/.local/bin/my-tool"     # make my-tool always go through the wrapper
    target: my-wrapper
    user: "${USER}"
```

### Write a config file inline

Use `write:` — never shell heredoc:

```yaml
tasks:
  - write: /etc/my-app/config.yml
    mode: "0644"
    user: root
    content: |
      listen: 0.0.0.0:8080
      log_level: info
      backends:
        - http://localhost:9090
```

### Add a service

Declare `service:` with a supervisord `[program:<name>]` fragment and add `supervisord` to `depends:`. The generator assembles per-layer service fragments into a single `/etc/supervisord.conf` at image build time.

---

## Style Guide

- Lowercase-hyphenated names for layers.
- System packages in `layer.yml` `rpm:`/`deb:`/`pac:`/`aur:` sections — not in `cmd:`.
- Python in `pixi.toml`, npm in `package.json`, Rust in `Cargo.toml`. Never `pip install` / `conda install` / `dnf install python3-*`.
- Binary downloads via `download:` verb; use `${ARCH}` or `${BUILD_ARCH}` for multi-arch URL templates.
- Never `dnf clean all` / `pacman -Scc` inside a `cmd:` — cache mounts handle it.
- Prefer `mkdir:` + `copy:` + `link:` + `write:` over `cmd:`. Fall back to `cmd:` only for true escape-hatch logic (git clone + cargo build, complex conditionals, etc.).
- Tasks are order-sensitive — put `download:` / `mkdir:` before the `copy:` / `write:` / `cmd:` that depends on them.
- No YAML anchors: write every task explicitly (same shape for `user: root` and `user: "${USER}"`).

---

## Cross-References

- `/ov:image` — Adding layers to image definitions; image composition; `data_image:` for data-only bundles; the full MCP-first authoring table including `image set`, `image add-layer`, `image rm-layer`, `image write`, `image cat`.
- `/ov:mcp` — "Authoring tools" table exposing `layer.set`, `layer.add-rpm`, `layer.add-deb`, `layer.add-pac`, `layer.add-aur` as MCP tools; end-to-end build-from-scratch worked example.
- `/ov:generate` — What `ov image generate` actually emits; the per-verb emitter pipeline; `.build/<image>/` layout.
- `/ov:validate` — Validation rules (including per-verb task requirements).
- `/ov:new` — Scaffolding a new layer directory.
- `/ov:build` — Building images (`--no-cache` caveat; multi-stage scratch).
- `/ov:config` — Cross-container `env_provides` / `mcp_provides` injection; `env_requires` enforcement; `--update-all`; resource caps.
- `/ov:deploy` — `deploy.yml` `provides:` section; tunnel is deploy.yml-only.
- `/ov:test` — `tests:` field for declarative layer checks (file/port/http/...); embedded in the `org.overthinkos.tests` OCI label under the `layer` section. Layer tests default to `scope: build`; opt into `scope: deploy` to reference runtime vars like `${HOST_PORT:N}`. **Cross-distro package tests:** use `package_map:` on a `package:` check to resolve distro-specific package names (Fedora `openssh-server` vs Arch `openssh`); see the skill's "Cross-distro package names" section and the worked example in `layers/sshd/layer.yml`.
- `/ov:sidecar` — Sidecars as `env_provides` participants (tailscale `TS_*` filtering).
- `/ov:secrets` — Credential store chain for `secret_accepts` / `secret_requires`.
- `/ov-layers:chrome` — Canonical consumer of `env_accepts` (proxy vars), resource caps (crash-loop circuit breaker), and heavy user-phase copy/mkdir task list.
- `/ov-layers:supervisord` — Event listener pattern triggered by resource caps.
- `/ov-layers:ov` — The ov-binary layer (composed by every ov-driving image). Paired with `/ov-layers:ov-mcp` which turns any image into an MCP server exposing the full ov CLI.
- `/ov-layers:ov-mcp` — Reference implementation of a meta-layer composition (`layers: [ov, supervisord]` — no install of its own, just wiring) with bind-mounted project directory and `OV_PROJECT_DIR` env-var plumbing.
- `/ov-layers:notebook-templates` — Data-layer example.
- `/ov-dev:generate` — Internal architecture of the task emission pipeline (Go side).

---

## When to Use This Skill

**MUST be invoked** for any task involving layer authoring, `layer.yml`, `tasks:`, `vars:`, `pixi.toml`, `package.json`, `Cargo.toml`, or any file under `layers/`. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Pre-build. Author layers before adding them to images. See `/ov:image` (composition), `/ov:build` (building), `/ov:generate` (emission internals).
