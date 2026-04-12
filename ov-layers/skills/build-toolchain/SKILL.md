---
name: build-toolchain
description: |
  C/C++ build toolchain with gcc, cmake, autoconf, ninja, git, and pkg-config.
  Use when working with native compilation, build tools, or C/C++ development.
---

# build-toolchain -- C/C++ compilation toolchain

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |

## Environment Variables

| Variable | Value |
|----------|-------|
| `CCACHE_DISABLE` | `1` |

## Packages

RPM: `autoconf`, `automake`, `binutils`, `bison`, `ccache`, `cli11-devel`, `cmake`, `flex`, `gcc`, `gcc-c++`, `gdb`, `git`, `glibc-devel`, `libtool`, `make`, `ninja-build`, `pkgconf`, `pkgconf-m4`, `pkgconf-pkg-config`, `redhat-rpm-config`, `rpm-build`, `rpm-sign`

## Usage

```yaml
# images.yml
my-image:
  layers:
    - build-toolchain
```

## Used In Images

- `/ov-images:fedora-builder`
- `/ov-images:archlinux-builder`
- `/ov-images:fedora-remote` (via remote layer ref)
- Also used in `bazzite-ai` (disabled)

## Related Layers
- `/ov-layers:pixi` — Sibling in builder images for conda-forge package builds
- `/ov-layers:nodejs` — Sibling in builder images for npm package builds
- `/ov-layers:yay` — Pairs in archlinux-builder for AUR builds

## Related Commands
- `/ov:build` — Multi-stage builders consume this layer for pixi/npm/cargo builds
- `/ov:generate` — See how builder stages are generated from this layer

## When to Use This Skill

Use when the user asks about:

- C/C++ compilation inside containers
- Build tools (cmake, make, ninja, autoconf)
- Compiler toolchain setup
- The `build-toolchain` layer or its packages
