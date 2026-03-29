---
name: arch-pac-test
description: |
  Test layer for pacman package installation on Arch Linux.
  Use when working with the arch-pac-test layer.
---

# arch-pac-test -- Arch pacman package test

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |
| Depends | none |

## Packages

PAC: `neovim`, `ripgrep`

## What It Does

Validates that the `pac:` package format works correctly on Arch Linux images. Installs neovim and ripgrep via pacman. Used as a smoke test for the multi-distro build system's Arch support.

## Usage

```yaml
# images.yml
layers:
  - arch-pac-test
```

Requires an Arch-based image with `build: [pac]` (or `[pac, aur]`).

## Included In

- `/ov-images:arch-test` — Arch packaging test image

## Related Layers

- `/ov-layers:arch-aur-test` — companion AUR package test

## When to Use This Skill

Use when the user asks about:
- Arch Linux pacman package testing
- The `pac:` package format in layer.yml
- The `arch-pac-test` layer
