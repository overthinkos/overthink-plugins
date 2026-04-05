---
name: list
description: |
  List components from images.yml and filesystem.
  MUST be invoked before any work involving: ov list commands, enumerating images, layers, build targets, services, routes, volumes, or aliases.
---

# ov list -- List Components

## Overview

Enumerate images, layers, build targets, and layer properties from `images.yml` and the filesystem. Useful for discovery and scripting.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| List images | `ov list images` | All images defined in images.yml |
| List layers | `ov list layers` | All layers found on filesystem |
| List targets | `ov list targets` | Build targets in dependency order |
| List services | `ov list services` | Layers that declare a `service:` field |
| List routes | `ov list routes` | Layers that declare a `route:` field |
| List volumes | `ov list volumes` | Layers that declare `volumes:` |
| List aliases | `ov list aliases` | Layers that declare `aliases:` |

## Usage

### List All Images

```bash
# Show all images from images.yml
ov list images

# Output: one image name per line
# fedora
# fedora-builder
# sway-browser-vnc
# ...
```

### List All Layers

```bash
# Show all layers found in layers/ directory
ov list layers
```

### Build Targets in Order

```bash
# Dependency-ordered list -- useful for scripting builds
ov list targets
```

### Filter by Property

```bash
# Which layers provide services?
ov list services

# Which layers define routes (Traefik)?
ov list routes

# Which layers declare persistent volumes?
ov list volumes

# Which layers provide host command aliases?
ov list aliases
```

## Cross-References

- `/ov:inspect` -- detailed inspection of a specific image
- `/ov:validate` -- validate images.yml and layer definitions
- `/ov:image` -- image definitions and inheritance
- `/ov:layer` -- layer authoring and structure
