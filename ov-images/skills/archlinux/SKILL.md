---
name: archlinux
description: |
  Base Arch Linux image. Root of the image hierarchy for all pac-based images.
  MUST be invoked before building, deploying, configuring, or troubleshooting the archlinux image.
---

# archlinux

Root base image built from `docker.io/library/archlinux:latest`. Foundation for all Arch Linux-based Overthink images.

## Image Properties

| Property | Value |
|----------|-------|
| Base | docker.io/library/archlinux:latest |
| Layers | (none) |
| Platforms | linux/amd64 |
| Distro | archlinux |
| Build | pac |
| Builders | pixi, npm, cargo, aur → archlinux-builder |
| Registry | ghcr.io/overthinkos |

## Quick Start

```bash
ov image build archlinux
ov shell archlinux
```

## Derived Images

- `/ov-images:archlinux-builder` — adds pixi, nodejs, build-toolchain, yay
- `/ov-images:arch-test` — pacman + AUR packaging test
- `/ov-images:arch-ov` — full ov toolchain on Arch

## Multi-Distro Support

This is the Arch counterpart to `/ov-images:fedora`. The tag system (`distro: [archlinux]`, `build: [pac]`) selects `pac:` package sections and `pac:` tasks in tasks:. Layers shared between Arch and Fedora images use distro-specific sections:

```yaml
# layer.yml — multi-distro package declarations
rpm:
  packages: [neovim]     # Fedora
pac:
  packages: [neovim]     # Arch
```

## Verification

After `ov image build`:
- `ov image list` — image appears in list
- `ov shell archlinux` — interactive shell works
- `ov shell archlinux -c "pacman --version"` — pacman available

## Related Images
- `/ov-images:archlinux-builder` — adds pixi, nodejs, build-toolchain, yay
- `/ov-images:arch-ov` — full ov toolchain on Arch
- `/ov-images:arch-test` — pacman + AUR test image
- `/ov-images:fedora` — RPM-based counterpart

## Related Commands
- `/ov:build` — build the archlinux base image
- `/ov:shell` — interactive shell in the base image

## When to Use This Skill

**MUST be invoked** when the task involves the archlinux base image or understanding the Arch image chain. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
