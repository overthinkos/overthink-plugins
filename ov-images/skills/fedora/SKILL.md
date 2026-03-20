---
name: fedora
description: |
  Base Fedora 43 image. Root of the image hierarchy for all RPM-based images.
  Use when working with the fedora base image or understanding the image chain.
---

# fedora

Root base image built from `quay.io/fedora/fedora:43`. Foundation for all RPM-based Overthink images.

## Image Properties

| Property | Value |
|----------|-------|
| Base | quay.io/fedora/fedora:43 |
| Layers | (none) |
| Platforms | linux/amd64 |
| Pkg | rpm |
| Registry | ghcr.io/overthinkos |

## Quick Start

```bash
ov build fedora
ov shell fedora
```

## Derived Images

- `/ov-images:fedora-nonfree` — adds RPM Fusion repos
- `/ov-images:fedora-builder` — adds pixi, nodejs, build-toolchain
- `/ov-images:nvidia` — adds CUDA toolkit
- `/ov-images:openclaw` — adds OpenClaw gateway
- `/ov-images:openclaw-sway-browser` — adds OpenClaw + desktop
- `/ov-images:githubrunner` — adds GitHub Actions runner

## Verification

After `ov build`:
- `ov list` — image appears in list
- `ov shell fedora` — interactive shell works

## When to Use This Skill

Use when the user asks about the fedora base image, the image inheritance chain, or the root of the build hierarchy.
