---
name: fedora-builder
description: |
  Builder image with pixi, Node.js, and C/C++ build toolchain. Used as the
  default builder for multi-stage image builds.
  Use when working with the builder image or build cache configuration.
---

# fedora-builder

Builder image with package managers and compilation tools. Default `builder` for all multi-stage image builds.

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

This image is referenced as `builder: fedora-builder` in `images.yml` defaults. The generator uses it as the first stage in multi-stage builds, providing build tools without bloating final images.

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

Use when the user asks about the builder image, multi-stage builds, build cache, or what tools are available during the build phase.
