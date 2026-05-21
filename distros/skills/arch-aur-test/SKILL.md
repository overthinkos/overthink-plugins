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

Validates that the `aur:` package format works correctly on Arch Linux images. Installs the `yay-bin` package from the AUR. Requires a builder image with `builds: [aur]` capability (i.e., `arch-builder` which includes the `yay` layer).

## Usage

```yaml
# image.yml
layers:
  - arch-aur-test
```

Requires an Arch-based image with `build: [pac, aur]` so the builder can resolve AUR packages.

## Used In Images

- `/ov-distros:arch-test` — Arch packaging test image

## Related Layers

- `/ov-distros:arch-pac-test` — companion pacman package test
- `/ov-tools:yay` — AUR helper that enables `aur:` builds

## When to Use This Skill

Use when the user asks about:
- Arch Linux AUR package testing
- The `aur:` package format in layer.yml
- The `arch-aur-test` layer

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
