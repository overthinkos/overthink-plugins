---
name: arch-test
description: |
  Test image for Arch Linux pacman and AUR package installation.
  MUST be invoked before building or troubleshooting the arch-test image.
---

# arch-test

Test image that validates both `pac:` (pacman) and `aur:` (AUR) package formats on Arch Linux. Used to verify the multi-distro build system handles Arch packaging correctly.

## Image Properties

| Property | Value |
|----------|-------|
| Base | archlinux |
| Build | pac, aur |
| Layers | agent-forwarding, arch-pac-test, arch-aur-test |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## What It Tests

- **`pac:` format** (via arch-pac-test) — installs `neovim` and `ripgrep` from official Arch repos
- **`aur:` format** (via arch-aur-test) — installs `yay-bin` from the AUR using archlinux-builder

## Quick Start

```bash
ov image build arch-test
ov shell arch-test
```

## Verification

```bash
ov shell arch-test -c "nvim --version"     # pac: package installed
ov shell arch-test -c "rg --version"       # pac: package installed
ov shell arch-test -c "which yay"          # aur: package installed
```

## Related

- `/ov-images:archlinux` — base image
- `/ov-images:archlinux-builder` — builder image (provides yay for AUR builds)
- `/ov-layers:arch-pac-test` — pacman test layer
- `/ov-layers:arch-aur-test` — AUR test layer

## When to Use This Skill

**MUST be invoked** when the task involves building or troubleshooting the arch-test image, or validating Arch Linux package format support.

## Related

- `/ov:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov:build` — `build.yml` vocabulary (distros, builders, init-systems)
