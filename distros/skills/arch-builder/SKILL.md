---
name: arch-builder
description: |
  Arch Linux builder image with pixi, Node.js, build toolchain, and yay AUR helper.
  Default builder for pixi, npm, cargo, and aur multi-stage builds on Arch.
  MUST be invoked before building, deploying, configuring, or troubleshooting the arch-builder image.
---

# arch-builder

Builder image for Arch Linux multi-stage builds. Counterpart to `/ov-distros:fedora-builder` — provides the same build capabilities (pixi, npm, cargo) plus AUR support via yay.

## Image Properties

| Property | Value |
|----------|-------|
| Base | arch |
| Layers | pixi, nodejs, build-toolchain, yay |
| Builds | pixi, npm, cargo, aur |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. **arch** — base Arch Linux image
2. **pixi** — pixi package manager (conda-forge Python environments)
3. **nodejs** — Node.js and npm
4. **build-toolchain** — gcc, cmake, autoconf, ninja, git, pkg-config
5. **yay** — AUR helper (base-devel, git, yay binary)

## Role in Build System

When an Arch-based image has layers with `pixi.toml`, `package.json`, `Cargo.toml`, or `aur:` packages, the build system uses this image as the builder for multi-stage builds. Configured via the `arch` image's `builder:` field (a map of build-type → builder-image):

```yaml
# image.yml
arch:
  builder:
    pixi: arch-builder
    npm: arch-builder
    cargo: arch-builder
    aur: arch-builder
```

The builder definitions themselves (pixi/npm/cargo/aur) live in `build.yml`'s `builder:` section — the same word is used intentionally because both maps key on the same slot (the build-type name).

## Quick Start

```bash
ov image build arch-builder
ov shell arch-builder -c "pixi --version"
ov shell arch-builder -c "node --version"
ov shell arch-builder -c "yay --version"
```

## Related

- `/ov-distros:arch` — parent base image
- `/ov-distros:fedora-builder` — Fedora counterpart (same role, no AUR)
- `/ov-tools:yay` — AUR helper layer (unique to Arch)
- `/ov-languages:pixi` — pixi package manager layer
- `/ov-coder:nodejs` — Node.js layer
- `/ov-coder:build-toolchain` — C/C++ build tools layer

## Verification

```bash
ov shell arch-builder -c "pixi --version && node --version && yay --version && gcc --version"
```

## When to Use This Skill

**MUST be invoked** when the task involves the arch-builder image, Arch multi-stage builds, or AUR package building infrastructure. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov-image:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
