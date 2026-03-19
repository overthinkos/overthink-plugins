---
name: generate
description: |
  Containerfile generation: understanding ov generate output, multi-stage builds,
  intermediate images, and the .build/ directory. Use when debugging or understanding
  generated Containerfiles.
---

# Generate - Containerfile Generation

## Overview

`ov generate` reads `images.yml` and `layers/`, resolves dependency graphs, and writes Containerfiles to `.build/`. Generation is idempotent and `.build/` is disposable (gitignored). Understanding the generated output is essential for debugging build issues.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Generate all | `ov generate` | Write .build/ (Containerfiles) |
| Generate with tag | `ov generate --tag v1.0.0` | Override CalVer tag |
| List targets | `ov list targets` | Build targets in dependency order |
| Inspect config | `ov inspect <image>` | Show resolved config JSON |

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

1. **Multi-stage build stages** -- scratch stages per layer (`COPY layers/<layer>/ /`), pixi build stages (`FROM <builder>`), npm build stages, supervisord config assembly, traefik routes
2. **`FROM ${BASE_IMAGE}`** -- external bases get bootstrap (task, user/group at UID/GID, WORKDIR); internal bases get `USER root`
3. **Image metadata** -- consolidated `ENV` directives, `EXPOSE` ports, `org.overthinkos.*` labels
4. **COPY build artifacts** -- pixi environments, pixi binary, npm packages from build stages
5. **Per-layer install steps** -- for each layer: rpm/deb packages, `root.yml`, `Cargo.toml`, `user.yml`. `USER` toggles between root and UID
6. **Final assembly** -- supervisord config concatenation, traefik routes COPY, `USER <UID>`, `RUN bootc container lint` (bootc images only -- validates bootc compliance)

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

Built images embed runtime metadata as labels (prefix: `org.overthinkos.`), making images self-describing for runtime commands (`ov shell`, `ov start`, `ov enable`, `ov alias install`).

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
| `org.overthinkos.bind_mounts` | JSON | `[{"name":"secrets","path":"...","encrypted":true}]` |
| `org.overthinkos.security` | JSON | `{"privileged":false,"cap_add":["SYS_PTRACE"]}` |
| `org.overthinkos.network` | string | `"host"` (omitted if default) |
| `org.overthinkos.tunnel` | JSON | tunnel config (omitted if none) |
| `org.overthinkos.fqdn` | string | FQDN for tunnel routing |
| `org.overthinkos.acme_email` | string | ACME certificate email |
| `org.overthinkos.env` | JSON | `["KEY=VALUE"]` runtime env vars |
| `org.overthinkos.hooks` | JSON | lifecycle hooks config |
| `org.overthinkos.vm` | JSON | VM config (bootc images) |
| `org.overthinkos.libvirt` | JSON | libvirt XML snippets |
| `org.overthinkos.routes` | JSON | `[{"host":"app.localhost","port":8080}]` |
| `org.overthinkos.systemd` | JSON | systemd unit names (bootc) |
| `org.overthinkos.supervisord` | JSON | supervisord service names |
| `org.overthinkos.env_layers` | JSON | layer-level env vars (merged) |
| `org.overthinkos.path_append` | JSON | PATH append entries |

Volumes use short names in labels (prefix `ov-<image>-` added at runtime). Empty arrays are omitted. JSON built from sorted slices for cache stability. Runtime commands try `LoadConfig` (images.yml) first, falling back to `<engine> inspect` labels -- enabling `ov shell myimage` from any directory.

Source: `ov/labels.go`, `ov/generate.go` (`writeLabels`).

## Runtime-Only Features

Security configuration (`security:` in layer.yml/images.yml) and environment variable injection (`env:`, `env_file:`) are **runtime-only** features. They affect container run arguments (`--privileged`, `--cap-add`, `-e`) but do not appear in generated Containerfiles.

## Cache Mounts

| Step type | Cache path | Options |
|-----------|------------|---------|
| `rpm.packages`, `root.yml` (rpm) | `/var/cache/libdnf5` | `sharing=locked` |
| `deb.packages`, `root.yml` (deb) | `/var/cache/apt` + `/var/lib/apt` | `sharing=locked` |
| `user.yml` | `/tmp/npm-cache` | `uid=<UID>,gid=<GID>` |
| `Cargo.toml` | `/tmp/cargo-cache` | `uid=<UID>,gid=<GID>` |
| pixi build stage | `/tmp/pixi-cache` + `/tmp/rattler-cache` | `uid=<UID>,gid=<GID>` |
| npm build stage | `/tmp/npm-cache` | `uid=<UID>,gid=<GID>` |

UID/GID in cache mounts are dynamic (from resolved image config). All non-root cache mounts use flat `/tmp/<tool>-cache` paths to avoid buildah permission issues with nested paths.

## Common Workflows

### Debug Why a Build Fails

```bash
ov generate                              # Generate Containerfiles
cat .build/my-image/Containerfile        # Inspect the generated Containerfile
ov validate                              # Check for validation errors
ov inspect my-image                      # See resolved config
```

### Understand Layer Ordering

```bash
ov list targets                          # Shows dependency-ordered build sequence
ov inspect my-image --format layers      # Shows layer list for an image
```

## Cross-References

- `/ov-dev:go` -- Source code for the generator
- `/ov:validate` -- Validation rules
- `/ov:build` -- Building from generated Containerfiles
- Source: `ov/generate.go`, `ov/intermediates.go`, `ov/graph.go`

## When to Use This Skill

Use when the user asks about:

- Generated Containerfile contents or structure
- Multi-stage build architecture
- Intermediate images and cache optimization
- OCI labels and image metadata
- `.build/` directory contents
- "Why does the Containerfile look like X?"
