---
name: fedora-remote
description: |
  Fedora image using remote layer references from GitHub. Demonstrates
  the @github.com/org/repo/layers/name:version remote layer syntax.
  Use when working with remote layers or the fedora-remote image.
---

# fedora-remote

Fedora image built with remote layer references — layers fetched from GitHub at build time.

## Image Properties

| Property | Value |
|----------|-------|
| Base | quay.io/fedora/fedora:43 |
| Layers | (remote refs) |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## Remote Layer References

```yaml
layers:
  - "@github.com/overthinkos/overthink/layers/dev-tools:main"
  - "@github.com/overthinkos/overthink/layers/build-toolchain:v2026.64.1937"
  - "@github.com/overthinkos/overthink/layers/nodejs"
```

- `dev-tools:main` — pinned to main branch
- `build-toolchain:v2026.64.1937` — pinned to specific version tag
- `nodejs` — default branch (no version specified)

## Quick Start

```bash
ov build fedora-remote
ov shell fedora-remote
```

## Key Layers (remote)

- `/ov-layers:dev-tools` — bat, ripgrep, neovim, gh, direnv, etc.
- `/ov-layers:build-toolchain` — gcc, cmake, autoconf, ninja, git
- `/ov-layers:nodejs` — Node.js + npm

## Verification

After `ov build`:
- `ov list` — image appears in list
- `ov shell fedora-remote` — interactive shell works

## When to Use This Skill

Use when the user asks about the fedora-remote image, remote layer references, the `@github.com/...` layer syntax, or building images from external layer sources.
