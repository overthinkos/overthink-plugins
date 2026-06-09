---
name: arch
description: |
  Base Arch Linux image. Root of the image hierarchy for all pac-based images.
  MUST be invoked before building, deploying, configuring, or troubleshooting the arch image.
---

# arch

Root base image built from `quay.io/archlinux/archlinux`, pinned to a precise `base-*` date-serial tag in `base.yml` (the quay mirror has the same content as `docker.io/library/archlinux` without Docker Hub's pull-rate limit). Foundation for all Arch Linux-based OpenCharly images.

## Image Properties

| Property | Value |
|----------|-------|
| Base | quay.io/archlinux/archlinux (pinned `base-*` tag in base.yml) |
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

`arch` (this base) and `/charly-distros:arch-builder` **live in this
repo** (in the combined `base.yml`). The consumer Arch images live in the
**`overthinkos/arch`** repo (git submodule at **`image/arch`**), whose config is
`charly.yml` plus its per-kind sibling files (`box.yml`/`pod.yml`/`k8s.yml`,
and `vm.yml` where it has VMs), flat-imported via `import:`. It composes this repo's
layers by git reference and reaches `arch` / `arch-builder` by importing the
main repo under the `charly` namespace (`import: [{charly: ../..}]`), so its images
write `base: charly.arch` and route builders to `charly.arch-builder`:

- `/charly-distros:arch-builder` — adds pixi, nodejs, build-toolchain, yay (in this repo)
- `/charly-coder:arch-coder` — kitchen-sink dev image (in `image/arch`)
- `/charly-coder:charly-arch` — full charly toolchain on Arch (in `image/arch`)
- `/charly-distros:arch-test` — pacman + AUR packaging test (in `image/arch`)

## Multi-Distro Support

This is the Arch counterpart to `/charly-distros:fedora`. The tag system (`distro: [arch]`, `build: [pac]`) selects `pac:` package sections and `pac:` tasks in tasks:. Layers shared between Arch and Fedora images use distro-specific sections:

```yaml
# candy.yml — multi-distro package declarations
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
