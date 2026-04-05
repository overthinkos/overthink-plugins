---
name: inspect
description: |
  Image inspection showing resolved configuration as JSON.
  MUST be invoked before any work involving: ov inspect command, viewing image configuration, or querying image metadata.
---

# ov inspect -- Image Inspection

## Overview

Displays the fully resolved configuration of an image as JSON. Shows base image, layers, ports, platforms, registry, tags, builders, volumes, and all computed metadata.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Full inspect | `ov inspect <image>` | Show complete resolved config as JSON |
| Specific field | `ov inspect <image> --format FIELD` | Extract a specific field |

## Usage

```bash
# Inspect full image configuration
ov inspect fedora

# Get specific field
ov inspect jupyter --format ports

# Get the base image
ov inspect sway-browser-vnc --format base

# Get layer list
ov inspect ollama --format layers

# Get platforms
ov inspect fedora --format platforms
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

- `/ov:image` -- Image definitions in images.yml
- `/ov:list` -- List all available images
- `/ov:validate` -- Validate image and layer definitions
