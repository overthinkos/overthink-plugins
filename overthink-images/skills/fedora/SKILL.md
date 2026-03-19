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

- `/overthink-images:fedora-nonfree` — adds RPM Fusion repos
- `/overthink-images:fedora-builder` — adds pixi, nodejs, build-toolchain
- `/overthink-images:nvidia` — adds CUDA toolkit
- `/overthink-images:openclaw` — adds OpenClaw gateway
- `/overthink-images:openclaw-sway-browser` — adds OpenClaw + desktop
- `/overthink-images:githubrunner` — adds GitHub Actions runner

## When to Use This Skill

Use when the user asks about the fedora base image, the image inheritance chain, or the root of the build hierarchy.
