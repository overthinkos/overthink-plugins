---
name: generate
description: |
  Containerfile generation: understanding ov image generate output, multi-stage builds,
  intermediate images, and the .build/ directory. Use when debugging or understanding
  generated Containerfiles.
  MUST be invoked before reading or modifying any Go source file in ov/.
---

# Generate - Containerfile Generation

## Overview

`ov image generate` reads `images.yml` and `layers/`, resolves dependency graphs, and writes Containerfiles to `.build/`. Generation is idempotent and `.build/` is disposable (gitignored). Understanding the generated output is essential for debugging build issues.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Generate all | `ov image generate` | Write .build/ (Containerfiles) |
| Generate with tag | `ov image generate --tag v1.0.0` | Override CalVer tag |
| List targets | `ov image list targets` | Build targets in dependency order |
| Inspect config | `ov image inspect <image>` | Show resolved config JSON |

## Generated Output Structure

```
.build/
├── <image>/
│   ├── Containerfile              # One per image
│   ├── supervisor/*.conf          # Supervisord configs (if service layers)
│   └── traefik-routes.yml         # Traefik config (if route layers)
└── _layers/<name>                 # Symlinks to remote layers
```

## Containerfile Structure

The generated Containerfile follows this order:

1. **Multi-stage build stages** -- scratch stages per layer, builder stages from `builder.yml` templates (pixi, npm, aur, cargo), init system config assembly (driven by init.yml), traefik routes
2. **`FROM ${BASE_IMAGE}`** -- external bases get bootstrap from `distro.yml` (install cmd, cache mounts, workarounds); internal bases get `USER root`
3. **Image metadata** -- consolidated `ENV` directives, `EXPOSE` ports, `org.overthinkos.*` labels
4. **COPY build artifacts** -- config-driven from `builder.yml` `copy_artifacts` and `copy_binary` definitions
5. **Per-layer install steps** -- distro: override (first match) then build: formats (all in order). Install commands rendered from `distro.yml` format templates. `root.yml` (tag-based task dispatch), inline builders (cargo), `user.yml`. `USER` toggles between root and UID
6. **Final assembly** -- init system config assembly, traefik routes COPY, `USER <UID>`, `RUN bootc container lint` (bootc images only)

**Config-driven generation:** All format-specific install commands, cache mounts, repo setup, and builder stages are defined in `distro.yml` and `builder.yml` at the project root as Go `text/template` strings. Each distro entry in `distro.yml` contains both bootstrap config and its package format definitions. Referenced via `format_config:` in `images.yml` — supports local paths and remote `@github.com/org/repo/path:version` refs. Adding a new format (e.g., `apk` for Alpine) requires only YAML changes — zero Go code modifications.

## Multi-Stage Builds

### Pixi Build Stage

```dockerfile
FROM ghcr.io/overthinkos/fedora-builder:2026.48.1808 AS supervisord-pixi-build
WORKDIR /home/user
COPY layers/supervisord/pixi.toml pixi.toml
RUN pixi install
```

Uses the configured builder image. No `apt-get install` needed -- builder has pixi, node, npm, gcc, cmake, git.

### npm Build Stage

```dockerfile
FROM <builder> AS openclaw-npm-build
COPY layers/openclaw/package.json package.json
RUN npm install -g --prefix /npm-global
```

### AUR Build Stage (Arch Linux)

For layers with `aur:` packages, the generator creates a multi-stage build using the `builders.aur` image:

```dockerfile
FROM ghcr.io/overthinkos/archlinux-builder:2026.84.942 AS my-tool-aur-build
USER root
RUN echo 'user ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/builder
USER 1000
WORKDIR /home/user
RUN --mount=type=cache,dst=/var/cache/pacman/pkg,sharing=locked \
    mkdir -p /tmp/aur-build && \
    cp /etc/makepkg.conf /tmp/makepkg.conf && \
    sed -i '/^OPTIONS/s/ debug/ !debug/' /tmp/makepkg.conf && \
    yay -S --noconfirm --needed --builddir /tmp/aur-build --makepkgconf /tmp/makepkg.conf \
      aur-package && \
    mkdir -p /tmp/aur-pkgs && \
    find /tmp/aur-build -name '*.pkg.tar.zst' -exec cp {} /tmp/aur-pkgs/ \;
```

In the main image, the built packages are installed:
```dockerfile
COPY --from=my-tool-aur-build /tmp/aur-pkgs/ /tmp/aur-pkgs/
RUN --mount=type=cache,dst=/var/cache/pacman/pkg,sharing=locked \
    pacman -U --noconfirm /tmp/aur-pkgs/*.pkg.tar.zst && \
    rm -rf /tmp/aur-pkgs
```

Key details: passwordless sudo is required because yay calls `pacman -U` as root. Debug packages are disabled via a patched copy of `makepkg.conf` (the build user can't modify `/etc/` directly).

**Status:** Working. Verified with `yay-bin` in `arch-test` image.

### Scratch Context Stage

```dockerfile
FROM scratch AS mylib-ctx
COPY layers/mylib/ /
```

## Intermediate Images

Auto-generated at branch points where multiple images share common layer prefixes:

```
fedora (external)
  -> fedora-supervisord (auto: pixi + python + supervisord)
     -> fedora-test (adds: traefik, testapi)
     -> openclaw (adds: nodejs, openclaw)
```

`ComputeIntermediates()` uses a prefix trie to detect divergence points. `GlobalLayerOrder()` prioritizes popular layers for cache efficiency.

## User Resolution

Configurable via `user`, `uid`, `gid` fields in `images.yml` (defaults: `"user"`, 1000, 1000).

For external base images, `ov` calls `registry.go:InspectImageUser()` which:
1. Pulls the base image via go-containerregistry
2. Extracts `/etc/passwd` from image layers (top layer first)
3. Finds user at configured UID
4. If found: overrides username, home directory, and GID (e.g., `ubuntu` with home `/home/ubuntu` for `ubuntu:24.04`)
5. If not found: uses configured defaults, bootstrap creates the user

For internal base images, user context is inherited from the parent image.

The bootstrap conditionally creates the user:
```dockerfile
RUN getent passwd <UID> >/dev/null 2>&1 || \
    (getent group <GID> >/dev/null 2>&1 || groupadd -g <GID> <user> && \
     useradd -m -u <UID> -g <GID> -s /bin/bash <user>)
```

Source: `ov/registry.go` (`InspectImageUser`).

## OCI Labels

Built images embed runtime metadata as labels (prefix: `org.overthinkos.`), making images self-describing for runtime commands (`ov shell`, `ov start`, `ov config`, `ov alias install`).

| Label | Type | Example |
|-------|------|---------|
| `org.overthinkos.version` | string | `"1"` (schema version) |
| `org.overthinkos.image` | string | `"openclaw"` |
| `org.overthinkos.registry` | string | `"ghcr.io/overthinkos"` (omitted if empty) |
| `org.overthinkos.bootc` | string | `"true"` (omitted if false) |
| `org.overthinkos.uid` / `.gid` | string | `"1000"` |
| `org.overthinkos.user` / `.home` | string | `"user"` / `"/home/user"` |
| `org.overthinkos.ports` | JSON | `["18789:18789"]` |
| `org.overthinkos.volumes` | JSON | `[{"name":"data","path":"/home/user/.openclaw"}]` |
| `org.overthinkos.aliases` | JSON | `[{"name":"openclaw","command":"openclaw"}]` |
| `org.overthinkos.security` | JSON | `{"privileged":false,"cap_add":["SYS_PTRACE"]}` |
| `org.overthinkos.network` | string | `"host"` (omitted if default) |
| ~~`org.overthinkos.tunnel`~~ | | Removed — tunnel is a deploy-time concern (deploy.yml only) |
| `org.overthinkos.fqdn` | string | FQDN for tunnel routing |
| `org.overthinkos.acme_email` | string | ACME certificate email |
| `org.overthinkos.env` | JSON | `["KEY=VALUE"]` runtime env vars |
| `org.overthinkos.hooks` | JSON | lifecycle hooks config |
| `org.overthinkos.vm` | JSON | VM config (bootc images) |
| `org.overthinkos.libvirt` | JSON | libvirt XML snippets |
| `org.overthinkos.routes` | JSON | `[{"host":"app.localhost","port":8080}]` |
| `org.overthinkos.init` | string | active init system name |
| `org.overthinkos.services.<init>` | JSON | service names per init system |
| `org.overthinkos.env_layers` | JSON | layer-level env vars (merged) |
| `org.overthinkos.path_append` | JSON | PATH append entries |
| `org.overthinkos.engine` | string | Required run engine (`docker`/`podman`, omitted if any) |
| `org.overthinkos.port_protos` | JSON | `{"9222":"tcp"}` port protocol overrides (non-http only) |
| `org.overthinkos.port_relay` | JSON | `[9222]` ports with socat relay |
| `org.overthinkos.status` | string | Effective status: `working`, `testing`, or `broken` (always emitted) |
| `org.overthinkos.info` | string | Aggregated status info from image + non-working layers (omitted if empty) |
| `org.overthinkos.layer_versions` | JSON | `{"chrome":"2026.83.1430"}` layer name → CalVer (only versioned layers) |
| `org.overthinkos.skills` | string | Skill documentation URL (omitted if no skill exists) |

Volumes use short names in labels (prefix `ov-<image>-` added at runtime). Empty arrays are omitted. JSON built from sorted slices for cache stability. Runtime commands read OCI labels exclusively (via `ExtractMetadata` in `ov/labels.go`) plus `deploy.yml` overlay — they never touch `images.yml` at runtime. That's why `ov shell myimage` works from any directory as long as the image is in local storage (if not, `ExtractMetadata` returns `ErrImageNotLocal` and the CLI suggests `ov image pull`). See `/ov:image` for the build/deploy boundary and `/ov:pull` for the sentinel pattern. Labels also include `org.overthinkos.init` for init system identification and `org.overthinkos.services.<init>` for per-init service lists.

Source: `ov/labels.go`, `ov/generate.go` (`writeLabels`).

## Runtime-Only Features

Security configuration (`security:` in layer.yml/images.yml) and environment variable injection (`env:`, `env_file:`) are **runtime-only** features. They affect container run arguments (`--privileged`, `--cap-add`, `-e`) but do not appear in generated Containerfiles.

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

UID/GID in cache mounts are dynamic (from resolved image config). All non-root cache mounts use flat `/tmp/<tool>-cache` paths to avoid buildah permission issues with nested paths.

## Common Workflows

### Debug Why a Build Fails

```bash
ov image generate                              # Generate Containerfiles
cat .build/my-image/Containerfile        # Inspect the generated Containerfile
ov image validate                              # Check for validation errors
ov image inspect my-image                      # See resolved config
```

### Understand Layer Ordering

```bash
ov image list targets                          # Shows dependency-ordered build sequence
ov image inspect my-image --format layers      # Shows layer list for an image
```

## Cross-References

- `/ov-dev:go` -- Source code for the generator
- `/ov:validate` -- Validation rules
- `/ov:build` -- Building from generated Containerfiles
- Source: `ov/generate.go`, `ov/intermediates.go`, `ov/graph.go`

## When to Use This Skill

**MUST be invoked** before reading or modifying Go source files. Invoke this skill BEFORE launching Explore agents on ov/ code.
