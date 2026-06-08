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
| Install files | `candy.yml` (packages only) |
| Depends | none |

## Packages

AUR: `yay-bin`

## What It Does

Validates that the `aur:` package format works correctly on Arch Linux images. Installs the `yay-bin` package from the AUR. Requires a builder image with `builds: [aur]` capability (i.e., `arch-builder` which includes the `yay` layer).

## Usage

```yaml
# box.yml
layers:
  - arch-aur-test
```

Requires an Arch-based image with `build: [pac, aur]` so the builder can resolve AUR packages.

## Used In Images

- `/charly-distros:arch-test` — Arch packaging test image

## Related Layers

- `/charly-distros:arch-pac-test` — companion pacman package test
- `/charly-tools:yay` — AUR helper that enables `aur:` builds

## When to Use This Skill

Use when the user asks about:
- Arch Linux AUR package testing
- The `aur:` package format in candy.yml
- The `arch-aur-test` layer

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
