---
name: inspect
description: |
  Image inspection showing resolved configuration as JSON.
  MUST be invoked before any work involving: ov image inspect command, viewing image configuration, or querying image metadata.
---

# ov image inspect -- Image Inspection

Invoked as `ov image inspect <image>`. See `/ov:image` for the family overview.

## Overview

Displays the fully resolved configuration of an image as JSON. Shows base image, layers, ports, platforms, registry, tags, builders, volumes, and all computed metadata.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Full inspect | `ov image inspect <image>` | Show complete resolved config as JSON |
| Specific field | `ov image inspect <image> --format FIELD` | Extract a specific field |

## Usage

```bash
# Inspect full image configuration
ov image inspect fedora

# Get specific field
ov image inspect jupyter --format ports

# Get the base image
ov image inspect sway-browser-vnc --format base

# Get layer list
ov image inspect ollama --format layers

# Get platforms
ov image inspect fedora --format platforms
```

## Output Fields

The JSON output includes:

| Field | Description |
|-------|-------------|
| `name` | Image name |
| `base` | Base image (another image name or external reference) |
| `layers` | Ordered list of layers applied |
| `ports` | Exposed ports from all layers |
| `platforms` | Target platforms (e.g., `linux/amd64`, `linux/arm64`) |
| `registry` | Container registry for push |
| `tags` | Image tags |
| `builders` | Builder capabilities (pixi, npm, cargo) |
| `volumes` | Declared volumes from layers |
| `distro` | Distro identity tags |
| `build` | Package format tags |

## Cross-References

### `ov image` family siblings

- `/ov:image` -- Family overview + images.yml composition reference
- `/ov:build` -- Build the inspected image
- `/ov:generate` -- Containerfile generation for the inspected image
- `/ov:list` -- Enumerate images before inspecting one
- `/ov:merge` -- Post-build layer consolidation
- `/ov:new` -- Scaffold new layers
- `/ov:pull` -- Fetch prebuilt images into local storage
- `/ov:validate` -- Validate image and layer definitions before inspect

### Related skills

- `/ov:layer` -- Layer-level detail shown in inspect output
- `/ov:deploy` -- `deploy.yml` overlay applied on top of inspect's resolved config
