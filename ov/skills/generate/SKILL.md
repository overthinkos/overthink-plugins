---
name: generate
description: |
  Containerfile generation from images.yml and layers.
  MUST be invoked before any work involving: ov image generate command, Containerfile generation, .build/ directory contents, or understanding generated output.
---

# ov image generate -- Containerfile Generation

Invoked as `ov image generate`. See `/ov:image` for the family overview.

## Overview

Parses `images.yml`, scans `layers/`, resolves the dependency graph, and generates all build artifacts into the `.build/` directory. Called internally by `ov image build` but can be run standalone to inspect generated output.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Generate all | `ov image generate` | Generate Containerfiles for all enabled images |
| With tag | `ov image generate --tag TAG` | Override the image tag |
| Single image | `ov image generate <image>` | Generate for a specific image |

## Usage

```bash
# Generate all Containerfiles
ov image generate

# Generate with a custom tag
ov image generate --tag v1.2.3

# Generate for a specific image
ov image generate fedora

# Inspect the generated output
cat .build/fedora/Containerfile
```

## Generated Output

The `.build/` directory contains:

| Path | Description |
|------|-------------|
| `.build/<image>/Containerfile` | Generated Containerfile with unconditional RUN steps |
| `.build/<image>/traefik-routes.yml` | Traefik dynamic config (only for images with `route` layers) |
| `.build/<image>/<fragment_dir>/*.conf` | Init system service configs (e.g., `supervisor/`, `systemd/`) |
| `.build/_layers/<name>` | Symlinks to remote layer directories |

All generated files start with `# <path> (generated -- do not edit)`.

## Behavior

- Generation is idempotent -- safe to run repeatedly
- `.build/` is disposable and gitignored
- Resolves layer dependencies, validates configs, renders templates from `init.yml`
- Injects pixi manylinux fix into `pixi.toml` files during build
- Multi-stage builds use builder images defined in `builder.yml`

## Cross-References

### `ov image` family siblings

- `/ov:image` -- Family overview + images.yml composition reference
- `/ov:build` -- Building images (calls generate internally)
- `/ov:inspect` -- Inspect generated output for a specific image
- `/ov:list` -- Enumerate targets before generation
- `/ov:merge` -- Post-build layer consolidation
- `/ov:new` -- Scaffold a new layer to generate into
- `/ov:pull` -- Pull prebuilt images (orthogonal to generate)
- `/ov:validate` -- Validation rules for images and layers

### Related skills

- `/ov:layer` -- Layer authoring and layer.yml format
- `/ov-dev:generate` -- Deep dive on Containerfile emission internals and `builderRefForFormat`
- `/ov-dev:go` -- `LoadConfig`, `ExtractMetadata`, `ErrImageNotLocal` source locations
