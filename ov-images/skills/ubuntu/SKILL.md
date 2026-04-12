---
name: ubuntu
description: |
  Base Ubuntu 24.04 image. Currently disabled. Placeholder for Debian-based builds.
  MUST be invoked before building or troubleshooting the ubuntu image.
---

# ubuntu

Base Ubuntu 24.04 image for Debian-based container builds.

## Image Properties

| Property | Value |
|----------|-------|
| Base | ubuntu:24.04 |
| Pkg | deb |
| Layers | (none) |
| Platforms | linux/amd64, linux/arm64 |
| Status | **disabled** (set `enabled: true` in images.yml) |

## Notes

Placeholder image for testing the Debian (`deb`) package manager path. No layers defined — serves as a base for Debian-based image chains, analogous to how `fedora` serves as the RPM base.

## Related Images
- `/ov-images:debian` — sibling Debian-family base (also disabled)
- `/ov-images:fedora` — RPM-family base counterpart
- `/ov-images:archlinux` — pacman-family base counterpart

## Related Commands
- `/ov:build` — build the ubuntu base image
- `/ov:shell` — interactive shell in the base image

## When to Use This Skill

**MUST be invoked** when the task involves the ubuntu base image or Debian package manager support.
