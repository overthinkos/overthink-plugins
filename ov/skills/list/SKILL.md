---
name: list
description: |
  List components from images.yml and filesystem.
  MUST be invoked before any work involving: ov image list commands, enumerating images, layers, build targets, services, routes, volumes, or aliases.
---

# ov image list -- List Components

Invoked as `ov image list {images,layers,targets,services,routes,volumes,aliases}`.
See `/ov:image` for the family overview.

## Overview

Enumerate images, layers, build targets, and layer properties from `images.yml` and the filesystem. Useful for discovery and scripting.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| List images | `ov image list images` | All images defined in images.yml |
| List layers | `ov image list layers` | All layers found on filesystem |
| List targets | `ov image list targets` | Build targets in dependency order |
| List services | `ov image list services` | Layers that declare a `service:` field |
| List routes | `ov image list routes` | Layers that declare a `route:` field |
| List volumes | `ov image list volumes` | Layers that declare `volumes:` |
| List aliases | `ov image list aliases` | Layers that declare `aliases:` |

## Usage

### List All Images

```bash
# Show all images from images.yml
ov image list images

# Output: one image name per line
# fedora
# fedora-builder
# sway-browser-vnc
# ...
```

### List All Layers

```bash
# Show all layers found in layers/ directory
ov image list layers
```

### Build Targets in Order

```bash
# Dependency-ordered list -- useful for scripting builds
ov image list targets
```

### Filter by Property

```bash
# Which layers provide services?
ov image list services

# Which layers define routes (Traefik)?
ov image list routes

# Which layers declare persistent volumes?
ov image list volumes

# Which layers provide host command aliases?
ov image list aliases
```

## Cross-References

### `ov image` family siblings

- `/ov:image` -- Family overview + images.yml composition reference
- `/ov:build` -- Build enumerated images
- `/ov:generate` -- Containerfile generation
- `/ov:inspect` -- Detailed inspection of a specific image
- `/ov:merge` -- Post-build layer consolidation
- `/ov:new` -- Scaffold a new layer
- `/ov:pull` -- Pull prebuilt images into local storage
- `/ov:validate` -- Validate images.yml and layer definitions

### Related skills

- `/ov:layer` -- Layer authoring and structure
