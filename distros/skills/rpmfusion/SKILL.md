---
name: rpmfusion
description: |
  RPM Fusion free and nonfree repository configuration for Fedora.
  Use when working with RPM Fusion repos, multimedia codecs, or nonfree packages.
---

# rpmfusion -- RPM Fusion repository configuration

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `tasks:` |

## Usage

```yaml
# image.yml
fedora-nonfree:
  base: fedora
  layers:
    - rpmfusion
```

## Used In Images

- `/ov-distros:fedora-nonfree` (base for immich and other images needing nonfree packages)
- `/ov-distros:fedora-builder` — applied **before** `build-toolchain` so cargo crates that
  need RPM Fusion free system libs (`libva-devel`, `x264-devel`, `ffmpeg-devel`) can install
  via dnf in the builder stage. Required for building pixelflux from source — see
  `/ov-selkies:selkies` (Patched pixelflux build pipeline).

## Related Layers
- `/ov-selkies:ffmpeg` -- nonfree multimedia codecs installed from RPM Fusion
- `/ov-immich:immich` -- consumes nonfree codecs via fedora-nonfree

## Related Commands
- `/ov-build:build` -- build the fedora-nonfree base image
- `/ov-image:image` -- inspect base/image inheritance

## When to Use This Skill

Use when the user asks about:

- RPM Fusion repository setup
- Nonfree or multimedia packages on Fedora
- The `fedora-nonfree` base image
- Enabling additional Fedora repositories

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
