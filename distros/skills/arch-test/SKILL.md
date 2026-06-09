---
name: arch-test
description: |
  Test image for Arch Linux pacman and AUR package installation.
  MUST be invoked before building or troubleshooting the arch-test image.
---

# arch-test

Test image that validates both `pac:` (pacman) and `aur:` (AUR) package formats on Arch Linux. Used to verify the multi-distro build system handles Arch packaging correctly.

`arch-test` lives in the **`overthinkos/arch`** repo (git submodule at
**`image/arch`**). The test layers `arch-pac-test` / `arch-aur-test` are vendored
locally in this repo's `candy/` (resolved via its `discover:` block); the `arch`
base + `arch-builder` come from main's `base.yml`, reached via the submodule's
`import:` of the main repo under the `charly` namespace — `base: charly.arch`. Build from
the submodule: `cd image/arch && charly box build arch-test`.

## Image Properties

| Property | Value |
|----------|-------|
| Base | arch |
| Build | pac, aur |
| Layers | agent-forwarding, arch-pac-test, arch-aur-test |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## What It Tests

- **`pac:` format** (via arch-pac-test) — installs `neovim` and `ripgrep` from official Arch repos
- **`aur:` format** (via arch-aur-test) — installs `yay-bin` from the AUR using arch-builder

## Quick Start

```bash
charly box build arch-test
charly shell arch-test
```

## Verification

```bash
charly shell arch-test -c "nvim --version"     # pac: package installed
charly shell arch-test -c "rg --version"       # pac: package installed
charly shell arch-test -c "which yay"          # aur: package installed
```

## Related

- `/charly-distros:arch` — base image
- `/charly-distros:arch-builder` — builder image (provides yay for AUR builds)
- `/charly-distros:arch-pac-test` — pacman test layer
- `/charly-distros:arch-aur-test` — AUR test layer

## When to Use This Skill

**MUST be invoked** when the task involves building or troubleshooting the arch-test image, or validating Arch Linux package format support.

## Related

- `/charly-image:image` — image family umbrella (`image:` entries in `charly.yml`, build/validate/inspect/list)
- `/charly-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
