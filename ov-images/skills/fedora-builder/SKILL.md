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
| Layers | rpmfusion, pixi, nodejs, build-toolchain |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (quay.io/fedora/fedora:43)
2. `rpmfusion` — RPM Fusion free + nonfree repo configuration. Applied **first** so subsequent layers can `dnf install` packages from RPM Fusion (e.g. `x264-devel`, `ffmpeg-devel`, `libva-devel`)
3. `pixi` — pixi package manager + env paths
4. `nodejs` — Node.js + npm
5. `build-toolchain` — gcc, cmake, autoconf, ninja, git, pkg-config, **plus** the smithay/cargo build deps (`rust`, `cargo`, `clang-devel`, `nasm`, `wayland-devel`, `libva-devel`, `x264-devel`, `ffmpeg-devel`, `pixman-devel`, …) needed for builder-stage compilation of Wayland compositors like pixelflux. See `/ov-layers:build-toolchain` for the full grouped list.

## Why rpmfusion is first

The build-toolchain layer's `dnf install` references packages that live in RPM Fusion free
(`libva-devel`, `x264-devel`, `ffmpeg-devel`). These are required to build pixelflux from
source (its `libva-sys`, `x264-sys`, and `ffmpeg-sys-next` cargo crates link against the
system libs). Without rpmfusion applied first, those `dnf install` calls would fail.
See `/ov-layers:selkies` (Patched pixelflux build pipeline) for the consumer story.

## Role in Build System

This image is referenced in `defaults.builders` as the builder for `pixi`, `npm`, and `cargo` multi-stage builds. It declares `builds: [pixi, npm, cargo]` to advertise its capabilities. The generator uses it as the `FROM` stage in multi-stage builds, providing build tools without bloating final images.

## Quick Start

```bash
ov image build fedora-builder
ov shell fedora-builder
```

## Key Layers

- `/ov-layers:pixi` — Python package management foundation
- `/ov-layers:nodejs` — Node.js runtime and npm
- `/ov-layers:build-toolchain` — C/C++ compilation tools

## Related Images

- `/ov-images:fedora` — parent base

## Verification

After `ov image build`:
- `ov image list` — image appears in list
- `ov shell fedora-builder` — interactive shell works

## When to Use This Skill

**MUST be invoked** when the task involves the fedora-builder image, multi-stage builds, or build cache configuration. Invoke this skill BEFORE reading source code or launching Explore agents.
