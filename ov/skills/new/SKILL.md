---
name: new
description: |
  Scaffold new layers with template files.
  MUST be invoked before any work involving: ov new command, creating new layers, or scaffolding layer directories.
---

# ov new -- Scaffold New Layers

## Overview

Creates a new layer directory with a template `layer.yml` file. The scaffolded layer is ready to be customized with packages, install scripts, and configuration.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| New layer | `ov new layer <name>` | Create a new layer directory with template |

## Usage

```bash
# Create a new layer
ov new layer my-tool

# This creates:
# layers/my-tool/
# layers/my-tool/layer.yml
```

## Workflow

After scaffolding, the typical workflow is:

1. `ov new layer <name>` -- create the scaffold
2. Edit `layers/<name>/layer.yml` -- add packages, deps, description
3. Add install files (`root.yml`, `user.yml`, `pixi.toml`, etc.) as needed
4. Add the layer to an image in `images.yml`
5. `ov validate` -- check for errors
6. `ov build <image>` -- build the image

## Naming Rules

- Lowercase-hyphenated names only (e.g., `my-tool`, `desktop-apps`)
- Must not conflict with existing layer names

## Cross-References

- `/ov:layer` -- Layer authoring guide, layer.yml format, install files
- `/ov:validate` -- Validate layer and image definitions
- `/ov:build` -- Build images containing the new layer
