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

`arch` (this base) and `/ov-distros:arch-builder` **live in this
repo** (in the combined `base.yml`). The consumer Arch images live in the
**`overthinkos/arch`** repo (git submodule at **`image/arch`**), which is a
single `overthink.yml` (all per-kind entries inlined). It composes this repo's
layers by git reference and reaches `arch` / `arch-builder` by importing the
main repo under the `ov` namespace (`import: [{ov: ../..}]`), so its images
write `base: ov.arch` and route builders to `ov.arch-builder`:

- `/ov-distros:arch-builder` — adds pixi, nodejs, build-toolchain, yay (in this repo)
- `/ov-coder:arch-coder` — kitchen-sink dev image (in `image/arch`)
- `/ov-coder:arch-ov` — full ov toolchain on Arch (in `image/arch`)
- `/ov-distros:arch-test` — pacman + AUR packaging test (in `image/arch`)

## Multi-Distro Support

This is the Arch counterpart to `/ov-distros:fedora`. The tag system (`distro: [arch]`, `build: [pac]`) selects `pac:` package sections and `pac:` tasks in tasks:. Layers shared between Arch and Fedora images use distro-specific sections:

```yaml
# layer.yml — multi-distro package declarations
rpm:
  packages: [neovim]     # Fedora
pac:
  packages: [neovim]     # Arch
```

AUR packages are authored under `distro.arch.aur.package` and built via
`arch-builder` (`yay`); the consuming image must add `aur` to its `build:` list
(`build: [pac, aur]`) — the base declares only `build: [pac]`. The
`/ov-distros:cachyos` base shares this exact AUR path (it routes `aur` →
`arch-builder` too), so AUR support is identical on `arch` and `cachyos`. See
`/ov-image:layer` "AUR (`aur:`)" for the authoring reference.

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
