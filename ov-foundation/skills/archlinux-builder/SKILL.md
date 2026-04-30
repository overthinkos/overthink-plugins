---
name: archlinux-builder
description: |
  Arch Linux builder image with pixi, Node.js, build toolchain, and yay AUR helper.
  Default builder for pixi, npm, cargo, and aur multi-stage builds on Arch.
  MUST be invoked before building, deploying, configuring, or troubleshooting the archlinux-builder image.
---

# archlinux-builder

Builder image for Arch Linux multi-stage builds. Counterpart to `/ov-foundation:fedora-builder` — provides the same build capabilities (pixi, npm, cargo) plus AUR support via yay.

## Image Properties

| Property | Value |
|----------|-------|
| Base | archlinux |
| Layers | pixi, nodejs, build-toolchain, yay |
| Builds | pixi, npm, cargo, aur |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. **archlinux** — base Arch Linux image
2. **pixi** — pixi package manager (conda-forge Python environments)
3. **nodejs** — Node.js and npm
4. **build-toolchain** — gcc, cmake, autoconf, ninja, git, pkg-config
5. **yay** — AUR helper (base-devel, git, yay binary)

## Role in Build System

When an Arch-based image has layers with `pixi.toml`, `package.json`, `Cargo.toml`, or `aur:` packages, the build system uses this image as the builder for multi-stage builds. Configured via the `archlinux` image's `builder:` field (a map of build-type → builder-image):

```yaml
# image.yml
archlinux:
  builder:
    pixi: archlinux-builder
    npm: archlinux-builder
    cargo: archlinux-builder
    aur: archlinux-builder
```

The builder definitions themselves (pixi/npm/cargo/aur) live in `build.yml`'s `builder:` section — the same word is used intentionally because both maps key on the same slot (the build-type name).

## Quick Start

```bash
ov image build archlinux-builder
ov shell archlinux-builder -c "pixi --version"
ov shell archlinux-builder -c "node --version"
ov shell archlinux-builder -c "yay --version"
```

## Related

- `/ov-foundation:archlinux` — parent base image
- `/ov-foundation:fedora-builder` — Fedora counterpart (same role, no AUR)
- `/ov-foundation:yay` — AUR helper layer (unique to Arch)
- `/ov-foundation:pixi` — pixi package manager layer
- `/ov-coder:nodejs` — Node.js layer
- `/ov-coder:build-toolchain` — C/C++ build tools layer

## Verification

```bash
ov shell archlinux-builder -c "pixi --version && node --version && yay --version && gcc --version"
```

## When to Use This Skill

**MUST be invoked** when the task involves the archlinux-builder image, Arch multi-stage builds, or AUR package building infrastructure. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov-build:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
