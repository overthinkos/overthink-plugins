---
name: list
description: |
  List components from box.yml and filesystem.
  MUST be invoked before any work involving: ov box list commands, enumerating images, layers, build targets, services, routes, volumes, or aliases.
---

# ov box list -- List Components

Invoked as `ov box list {images,layers,targets,services,routes,volumes,aliases}`.
See `/ov-image:image` for the family overview.

## Overview

Enumerate images, layers, build targets, and layer properties from `box.yml` and the filesystem. Useful for discovery and scripting.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| List images | `ov box list boxes` | All images defined in box.yml |
| List layers | `ov box list candies` | All layers found on filesystem |
| List targets | `ov box list targets` | Build targets in dependency order |
| List services | `ov box list services` | Layers that declare a `service:` field |
| List routes | `ov box list routes` | Layers that declare a `route:` field |
| List volumes | `ov box list volumes` | Layers that declare `volume:` |
| List aliases | `ov box list aliases` | Layers that declare `alias:` |

## Usage

### List All Images

```bash
# Show all images from box.yml
ov box list boxes

# Output: one image name per line
# fedora
# fedora-builder
# sway-browser-vnc
# ...
```

### List All Layers

```bash
# Show all layers found in candy/ directory
ov box list candies
```

### Build Targets in Order

```bash
# Dependency-ordered list -- useful for scripting builds
ov box list targets
```

### Filter by Property

```bash
# Which layers provide services?
ov box list services

# Which layers define routes (Traefik)?
ov box list routes

# Which layers declare persistent volumes?
ov box list volumes

# Which layers provide host command aliases?
ov box list aliases
```

## Project directory override

`ov box list …` resolves `box.yml` + `candy/` via `os.Getwd()`. Override with `-C <dir>` / `--dir <dir>` / `OV_PROJECT_DIR=<dir>`. See `/ov-image:image` "Project directory resolution".

## Cross-References

### `ov box` family siblings

- `/ov-image:image` -- Family overview + box.yml composition reference
- `/ov-build:build` -- Build enumerated images
- `/ov-build:generate` -- Containerfile generation
- `/ov-build:inspect` -- Detailed inspection of a specific image
- `/ov-build:merge` -- Post-build layer consolidation
- `/ov-build:new` -- Scaffold a new layer
- `/ov-build:pull` -- Pull prebuilt images into local storage
- `/ov-build:validate` -- Validate box.yml and layer definitions

### Related skills

- `/ov-image:layer` -- Layer authoring and structure
