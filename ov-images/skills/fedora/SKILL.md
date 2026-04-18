---
name: fedora
description: |
  Base Fedora 43 image. Root of the image hierarchy for all RPM-based images.
  MUST be invoked before building, deploying, configuring, or troubleshooting the fedora image.
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
ov image build fedora
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

After `ov image build`:
- `ov image list` — image appears in list
- `ov shell fedora` — interactive shell works

## Related Images
- `/ov-images:fedora-nonfree` — adds RPM Fusion repos
- `/ov-images:fedora-builder` — adds pixi, nodejs, build-toolchain
- `/ov-images:fedora-ov` — full ov toolchain on Fedora
- `/ov-images:archlinux` — pacman-based counterpart base

## Related Commands
- `/ov:build` — build the fedora base image
- `/ov:shell` — interactive shell in the base image

## When to Use This Skill

**MUST be invoked** when the task involves the fedora base image or understanding the image chain. Invoke this skill BEFORE reading source code or launching Explore agents.
