---
name: list
description: |
  List components from image.yml and filesystem.
  MUST be invoked before any work involving: ov image list commands, enumerating images, layers, build targets, services, routes, volumes, or aliases.
---

# ov image list -- List Components

Invoked as `ov image list {images,layers,targets,services,routes,volumes,aliases}`.
See `/ov-build:image` for the family overview.

## Overview

Enumerate images, layers, build targets, and layer properties from `image.yml` and the filesystem. Useful for discovery and scripting.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| List images | `ov image list images` | All images defined in image.yml |
| List layers | `ov image list layers` | All layers found on filesystem |
| List targets | `ov image list targets` | Build targets in dependency order |
| List services | `ov image list services` | Layers that declare a `service:` field |
| List routes | `ov image list routes` | Layers that declare a `route:` field |
| List volumes | `ov image list volumes` | Layers that declare `volumes:` |
| List aliases | `ov image list aliases` | Layers that declare `aliases:` |

## Usage

### List All Images

```bash
# Show all images from image.yml
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

## Project directory override

`ov image list …` resolves `image.yml` + `layers/` via `os.Getwd()`. Override with `-C <dir>` / `--dir <dir>` / `OV_PROJECT_DIR=<dir>`. See `/ov-build:image` "Project directory resolution".

## Cross-References

### `ov image` family siblings

- `/ov-build:image` -- Family overview + image.yml composition reference
- `/ov-build:build` -- Build enumerated images
- `/ov-build:generate` -- Containerfile generation
- `/ov-build:inspect` -- Detailed inspection of a specific image
- `/ov-build:merge` -- Post-build layer consolidation
- `/ov-build:new` -- Scaffold a new layer
- `/ov-build:pull` -- Pull prebuilt images into local storage
- `/ov-build:validate` -- Validate image.yml and layer definitions

### Related skills

- `/ov-build:layer` -- Layer authoring and structure
