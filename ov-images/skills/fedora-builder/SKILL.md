---
name: fedora-builder
description: |
  Builder image with pixi, Node.js, and C/C++ build toolchain. Used as the
  default builder for multi-stage image builds.
  MUST be invoked before building, deploying, configuring, or troubleshooting the fedora-builder image.
---

# fedora-builder

Builder image with package managers and compilation tools. Default builder for pixi, npm, and cargo multi-stage builds (declared via `builds: [pixi, npm, cargo]`).

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | pixi, nodejs, build-toolchain |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (quay.io/fedora/fedora:43)
2. `pixi` — pixi package manager + env paths
3. `nodejs` — Node.js + npm
4. `build-toolchain` — gcc, cmake, autoconf, ninja, git, pkg-config

## Role in Build System

This image is referenced in `defaults.builders` as the builder for `pixi`, `npm`, and `cargo` multi-stage builds. It declares `builds: [pixi, npm, cargo]` to advertise its capabilities. The generator uses it as the `FROM` stage in multi-stage builds, providing build tools without bloating final images.

## Quick Start

```bash
ov build fedora-builder
ov shell fedora-builder
```

## Key Layers

- `/ov-layers:pixi` — Python package management foundation
- `/ov-layers:nodejs` — Node.js runtime and npm
- `/ov-layers:build-toolchain` — C/C++ compilation tools

## Related Images

- `/ov-images:fedora` — parent base

## Verification

After `ov build`:
- `ov list` — image appears in list
- `ov shell fedora-builder` — interactive shell works

## When to Use This Skill

**MUST be invoked** when the task involves the fedora-builder image, multi-stage builds, or build cache configuration. Invoke this skill BEFORE reading source code or launching Explore agents.
