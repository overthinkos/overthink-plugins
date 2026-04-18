---
name: new
description: |
  Scaffold new layers with template files.
  MUST be invoked before any work involving: ov image new layer command, creating new layers, or scaffolding layer directories.
---

# ov image new layer -- Scaffold New Layers

Invoked as `ov image new layer <name>`. See `/ov:image` for the family overview.

## Overview

Creates a new layer directory with a template `layer.yml` file. The scaffolded layer is ready to be customized with packages, install scripts, and configuration.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| New layer | `ov image new layer <name>` | Create a new layer directory with template |

## Usage

```bash
# Create a new layer
ov image new layer my-tool

# This creates:
# layers/my-tool/
# layers/my-tool/layer.yml
```

## Workflow

After scaffolding, the typical workflow is:

1. `ov image new layer <name>` — create the scaffold
2. Edit `layers/<name>/layer.yml` — add packages, deps, `tasks:` (see `/ov:layer` for the verb catalog)
3. Optionally add auto-detected package manifests (`pixi.toml`, `package.json`, `Cargo.toml`) if the layer ships Python / npm / Rust dependencies
4. Add the layer to an image in `images.yml`
5. `ov image validate` — check for errors
6. `ov image build <image>` — build the image

The scaffolded `layer.yml` is minimal — just `depends:` and a `description:`. Add sections as needed: `rpm:` / `deb:` / `pac:` / `aur:` for system packages, `env:` for runtime environment, `ports:` / `service:` / `volumes:` for services, and `tasks:` for install operations (mkdir, copy, write, download, link, setcap, cmd, build). The scaffolder no longer creates separate Taskfile shell scripts — all install logic flows through `tasks:` in `layer.yml`.

## Naming Rules

- Lowercase-hyphenated names only (e.g., `my-tool`, `desktop-apps`)
- Must not conflict with existing layer names

## Cross-References

### `ov image` family siblings

- `/ov:image` -- Family overview + images.yml composition reference
- `/ov:build` -- Build images containing the new layer
- `/ov:generate` -- Containerfile generation
- `/ov:inspect` -- Inspect built image including the new layer
- `/ov:list` -- Enumerate layers (the new one shows up here)
- `/ov:merge` -- Post-build layer consolidation
- `/ov:pull` -- Pull prebuilt images (unrelated; for consumers of your new layer)
- `/ov:validate` -- Validate layer and image definitions

### Related skills

- `/ov:layer` -- Layer authoring guide, layer.yml format, install files
