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
└── _layers/<name>                 # Symlinks to remote module layers
```

## Containerfile Structure

The generated Containerfile follows this order:

1. **Multi-stage build stages** -- scratch stages per layer (`COPY layers/<layer>/ /`), pixi build stages (`FROM <builder>`), npm build stages, supervisord config assembly, traefik routes
2. **`FROM ${BASE_IMAGE}`** -- external bases get bootstrap (task, user/group at UID/GID, WORKDIR); internal bases get `USER root`
3. **Image metadata** -- consolidated `ENV` directives, `EXPOSE` ports, `org.overthink.*` labels
4. **COPY build artifacts** -- pixi environments, pixi binary, npm packages from build stages
5. **Per-layer install steps** -- for each layer: rpm/deb packages, `root.yml`, `Cargo.toml`, `user.yml`. `USER` toggles between root and UID
6. **Final assembly** -- supervisord config concatenation, traefik routes COPY, `USER <UID>`, `RUN bootc container lint` (bootc only)

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

## OCI Labels

Built images embed runtime metadata as labels (prefix: `org.overthink.`):

| Label | Example |
|-------|---------|
| `org.overthink.image` | `"openclaw"` |
| `org.overthink.ports` | `["18789:18789"]` |
| `org.overthink.volumes` | `[{"name":"data","path":"/home/user/.openclaw"}]` |
| `org.overthink.aliases` | `[{"name":"openclaw","command":"openclaw"}]` |

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

- `/overthink-dev:go` -- Source code for the generator
- `/overthink-dev:validate` -- Validation rules
- `/overthink:build` -- Building from generated Containerfiles
- Source: `ov/generate.go`, `ov/intermediates.go`, `ov/graph.go`

## When to Use This Skill

Use when the user asks about:

- Generated Containerfile contents or structure
- Multi-stage build architecture
- Intermediate images and cache optimization
- OCI labels and image metadata
- `.build/` directory contents
- "Why does the Containerfile look like X?"
