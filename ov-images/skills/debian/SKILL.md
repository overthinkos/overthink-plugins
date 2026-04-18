---
name: debian
description: |
  Base Debian 13 image. Currently disabled. Placeholder for Debian-based builds.
  MUST be invoked before building or troubleshooting the debian image.
---

# debian

Base Debian 13 image for Debian-based container builds.

## Image Properties

| Property | Value |
|----------|-------|
| Base | debian:13 |
| Pkg | deb |
| Layers | (none) |
| Platforms | linux/amd64, linux/arm64 |
| Status | **disabled** (set `enabled: true` in image.yml) |

## Notes

Placeholder image for testing the Debian (`deb`) package manager path. No layers defined — serves as a potential alternative to the `ubuntu` base.

## Related Images
- `/ov-images:ubuntu` — sibling Debian-family base (also disabled)
- `/ov-images:fedora` — RPM-family base counterpart
- `/ov-images:archlinux` — pacman-family base counterpart

## Related Commands
- `/ov:build` — build the debian base image
- `/ov:shell` — interactive shell in the base image

## When to Use This Skill

**MUST be invoked** when the task involves the debian base image or Debian package support.
