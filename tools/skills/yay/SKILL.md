---
name: yay
description: |
  AUR helper for Arch Linux, enabling aur: package sections in candy.yml.
  Use when working with the yay layer or Arch AUR builds.
---

# yay -- AUR helper for Arch Linux

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `candy.yml`, `task:` |
| Depends | none |

## Packages

PAC: `base-devel`, `git`

## Install Script

The `task:` task downloads the latest `yay` binary from GitHub releases:

```yaml
# task: (in candy.yml)
task:
  all:
    cmds:
      - |
        ARCH=$(uname -m)
        URL=$(curl -fsSL https://api.github.com/repos/Jguer/yay/releases/latest \
          | grep -o "https://github.com/Jguer/yay/releases/download/[^\"]*_${ARCH}.tar.gz")
        curl -fsSL "$URL" | tar -xzf - -C /usr/local/bin --strip-components=1 --wildcards '*/yay'
```

Architecture-aware: downloads the correct binary for `x86_64` or `aarch64`.

## What It Does

Installs the `yay` AUR helper, which enables the `aur:` package format in `candy.yml`. Any layer with an `aur:` section requires a builder that has the `yay` layer (and `builds: [aur]` capability). The `base-devel` and `git` packages are prerequisites for building AUR packages.

## Usage

```yaml
# box.yml — typically in a builder image
layers:
  - yay
```

Not used directly in end-user images. Instead, it's part of the builder image that compiles AUR packages during multi-stage builds.

## Used In Images

- `/charly-distros:arch-builder` — Arch build infrastructure image

## Related Layers

- `/charly-coder:build-toolchain` — C/C++ build tools (also in arch-builder)
- `/charly-distros:arch-aur-test` — test layer that validates AUR builds

## When to Use This Skill

Use when the user asks about:
- AUR package support in OpenCharly
- The `yay` layer or AUR helper installation
- How `aur:` packages in candy.yml get built
- The `arch-builder` image's AUR capability

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
