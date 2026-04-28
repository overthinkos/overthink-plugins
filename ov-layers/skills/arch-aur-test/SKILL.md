---
name: arch-aur-test
description: |
  Test layer for AUR package installation on Arch Linux.
  Use when working with the arch-aur-test layer.
---

# arch-aur-test -- Arch AUR package test

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |
| Depends | none |

## Packages

AUR: `yay-bin`

## What It Does

Validates that the `aur:` package format works correctly on Arch Linux images. Installs the `yay-bin` package from the AUR. Requires a builder image with `builds: [aur]` capability (i.e., `archlinux-builder` which includes the `yay` layer).

## Usage

```yaml
# image.yml
layers:
  - arch-aur-test
```

Requires an Arch-based image with `build: [pac, aur]` so the builder can resolve AUR packages.

## Used In Images

- `/ov-images:arch-test` — Arch packaging test image

## Related Layers

- `/ov-layers:arch-pac-test` — companion pacman package test
- `/ov-layers:yay` — AUR helper that enables `aur:` builds

## When to Use This Skill

Use when the user asks about:
- Arch Linux AUR package testing
- The `aur:` package format in layer.yml
- The `arch-aur-test` layer

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
