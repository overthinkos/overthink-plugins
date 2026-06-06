---
name: inspect
description: |
  Image inspection showing resolved configuration as JSON.
  MUST be invoked before any work involving: ov box inspect command, viewing image configuration, or querying image metadata.
---

# ov box inspect -- Image Inspection

Invoked as `ov box inspect <image>`. See `/ov-image:image` for the family overview.

## Overview

Displays the fully resolved configuration of an image as JSON. Shows base image, layers, ports, platforms, registry, tags, builder selections, volumes, and all computed metadata.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Full inspect | `ov box inspect <image>` | Show complete resolved config as JSON |
| Specific field | `ov box inspect <image> --format FIELD` | Extract a specific field |
| Disabled image | `ov box inspect <image> --include-disabled` | Operate on `enabled: false` images without flipping authored config |

## Usage

```bash
# Inspect full image configuration
ov box inspect fedora

# Get specific field
ov box inspect jupyter --format ports

# Get the base image
ov box inspect sway-browser-vnc --format base

# Get layer list
ov box inspect ollama --format layers

# Get platforms
ov box inspect fedora --format platforms

# Get the builder map (build-type → builder image)
ov box inspect arch --format builder

# Get the builder capabilities this image declares
ov box inspect fedora-builder --format builds
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
| `builder` | Build-type → builder-image map (e.g. `{"pixi": "fedora-builder"}`) — resolved from image → base → defaults |
| `builds` | Builder capabilities this image declares (e.g. `[pixi, npm, cargo, aur]`) — not inherited |
| `volumes` | Declared volumes from layers |
| `distro` | Distro identity tags |
| `build` | Package format tags |

All `--format` values are the JSON field names from the `inspect` output. When passing `--format` to select a map (like `builder`) or list, the CLI prints one entry per line.

**Caveat — `--format bind_mounts`**: this one format option reads `deploy.yml` (not `box.yml`), because bind-mount backings are a deploy-time concept (`ov config --bind <volume>` writes them to `deploy.yml`). The output is display-only — no OCI label contamination, no build-mode state leak. All other `--format` values (`ports`, `volumes`, `layers`, `base`, `builder`, …) are strictly `box.yml`-derived per the mode-purity invariant (see `/ov-build:build` and `/ov-internals:go` "Mode purity").

## Project directory override

`ov box inspect` resolves `box.yml` via `os.Getwd()`. Override with `-C <dir>` / `--dir <dir>` / `OV_PROJECT_DIR=<dir>`. See `/ov-image:image` "Project directory resolution".

## Cross-References

### `ov box` family siblings

- `/ov-image:image` -- Family overview + box.yml composition reference
- `/ov-build:build` -- Build the inspected image
- `/ov-build:generate` -- Containerfile generation for the inspected image
- `/ov-build:list` -- Enumerate images before inspecting one
- `/ov-build:merge` -- Post-build layer consolidation
- `/ov-build:new` -- Scaffold new layers
- `/ov-build:pull` -- Fetch prebuilt images into local storage
- `/ov-build:validate` -- Validate image and layer definitions before inspect

### Related skills

- `/ov-image:layer` -- Layer-level detail shown in inspect output
- `/ov-core:deploy` -- `deploy.yml` overlay applied on top of inspect's resolved config
