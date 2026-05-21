---
name: arch
description: |
  Base Arch Linux image. Root of the image hierarchy for all pac-based images.
  MUST be invoked before building, deploying, configuring, or troubleshooting the arch image.
---

# arch

Root base image built from `docker.io/library/archlinux:latest`. Foundation for all Arch Linux-based Overthink images.

## Image Properties

| Property | Value |
|----------|-------|
| Base | docker.io/library/archlinux:latest |
| Layers | (none) |
| Platforms | linux/amd64 |
| Distro | arch |
| Build | pac |
| Builders | pixi, npm, cargo, aur → arch-builder |
| Registry | ghcr.io/overthinkos |

## Quick Start

```bash
ov image build arch
ov shell arch
```

## Derived Images

`arch` (this base) and `/ov-distros:arch-builder` **stay in this
repo**. The consumer Arch images were **relocated (2026-05)** to the
**`overthinkos/arch`** repo (git submodule at **`image/arch`**), where they
compose this repo's layers by git reference and pull `arch` /
`arch-builder` via a remote `include:` of `arch-base.yml`:

- `/ov-distros:arch-builder` — adds pixi, nodejs, build-toolchain, yay (stays here)
- `/ov-coder:arch-coder` — kitchen-sink dev image (now in `image/arch`)
- `/ov-coder:arch-ov` — full ov toolchain on Arch (now in `image/arch`)
- `/ov-distros:arch-test` — pacman + AUR packaging test (now in `image/arch`)

## Multi-Distro Support

This is the Arch counterpart to `/ov-distros:fedora`. The tag system (`distro: [arch]`, `build: [pac]`) selects `pac:` package sections and `pac:` tasks in tasks:. Layers shared between Arch and Fedora images use distro-specific sections:

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
- `ov shell arch` — interactive shell works
- `ov shell arch -c "pacman --version"` — pacman available

## Related Images
- `/ov-distros:arch-builder` — adds pixi, nodejs, build-toolchain, yay
- `/ov-coder:arch-ov` — full ov toolchain on Arch
- `/ov-distros:arch-test` — pacman + AUR test image
- `/ov-distros:fedora` — RPM-based counterpart

## Related Commands
- `/ov-build:build` — build the arch base image
- `/ov-core:shell` — interactive shell in the base image

## When to Use This Skill

**MUST be invoked** when the task involves the arch base image or understanding the Arch image chain. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov-image:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
