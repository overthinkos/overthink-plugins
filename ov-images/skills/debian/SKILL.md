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
| Status | **disabled** (set `enabled: true` in images.yml) |

## Notes

Placeholder image for testing the Debian (`deb`) package manager path. No layers defined — serves as a potential alternative to the `ubuntu` base.

## When to Use This Skill

**MUST be invoked** when the task involves the debian base image or Debian package support.
