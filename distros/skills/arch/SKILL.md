---
name: arch
description: |
  Base Arch Linux image. Root of the image hierarchy for all pac-based images.
  MUST be invoked before building, deploying, configuring, or troubleshooting the arch image.
---

# arch

Root base image built from `quay.io/archlinux/archlinux`, pinned to a precise `base-*` date-serial tag in the arch base box (the `overthinkos/arch` submodule at `box/arch`; the quay mirror has the same content as `docker.io/library/archlinux` without Docker Hub's pull-rate limit). Foundation for all Arch Linux-based OpenCharly images.

## Image Properties

| Property | Value |
|----------|-------|
| Base | quay.io/archlinux/archlinux (pinned `base-*` tag in the `overthinkos/arch` submodule) |
| Layers | (none) |
| Platforms | linux/amd64 |
| Distro | arch |
| Build | pac |
| Builders | pixi, npm, cargo, aur → arch-builder |
| Registry | ghcr.io/overthinkos |

## Quick Start

```bash
charly box build arch
charly shell arch
```

## Derived Images

`arch` (this base), `/charly-distros:arch-builder`, and the consumer Arch
images all live in the **`overthinkos/arch`** repo (git submodule at
**`box/arch`**), discovered as `box/<name>/charly.yml` boxes. That submodule
is SELF-CONTAINED (`import: []`): its base/builder stack is bare-local and it
composes the main repo's shared layers by `@github` git reference. Its images
write `base: arch` and route builders to `arch-builder` (bare-local refs, no
namespace qualifier):

- `/charly-distros:arch-builder` — adds pixi, nodejs, build-toolchain, yay (in `box/arch`)
- `/charly-coder:arch-coder` — kitchen-sink dev image (in `box/arch`)
- `/charly-coder:charly-arch` — full charly toolchain on Arch (in `box/arch`)
- `/charly-distros:arch-test` — pacman + AUR packaging test (in `box/arch`)

## Multi-Distro Support

This is the Arch counterpart to `/charly-distros:fedora`. The tag system (`distro: [arch]`, `build: [pac]`) selects `pac:` package sections and `pac:` tasks in tasks:. Layers shared between Arch and Fedora images use distro-specific sections:

```yaml
# charly.yml — multi-distro package declarations
rpm:
  packages: [neovim]     # Fedora
pac:
  packages: [neovim]     # Arch
```

AUR packages are authored under `distro.arch.aur.package` and built via
`arch-builder` (`yay`); the consuming image must add `aur` to its `build:` list
(`build: [pac, aur]`) — the base declares only `build: [pac]`. The
`/charly-distros:cachyos` base shares this exact AUR path (it routes `aur` →
`arch-builder` too), so AUR support is identical on `arch` and `cachyos`. See
`/charly-image:layer` "AUR (`aur:`)" for the authoring reference.

## Verification

After `charly box build`:
- `charly box list` — image appears in list
- `charly shell arch` — interactive shell works
- `charly shell arch -c "pacman --version"` — pacman available

## Related Images
- `/charly-distros:arch-builder` — adds pixi, nodejs, build-toolchain, yay
- `/charly-coder:charly-arch` — full charly toolchain on Arch
- `/charly-distros:arch-test` — pacman + AUR test image
- `/charly-distros:fedora` — RPM-based counterpart

## Related Commands
- `/charly-build:build` — build the arch base image
- `/charly-core:shell` — interactive shell in the base image

## When to Use This Skill

**MUST be invoked** when the task involves the arch base image or understanding the Arch image chain. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/charly-image:image` — image family umbrella (`image:` entries in `charly.yml`, build/validate/inspect/list)
