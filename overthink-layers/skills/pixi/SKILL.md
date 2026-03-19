---
name: pixi
description: |
  Pixi package manager binary with environment and PATH setup.
  Use when working with pixi, conda-forge packages, or Python environment management.
---

# pixi -- Pixi package manager

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `root.yml` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `PIXI_CACHE_DIR` | `~/.cache/pixi` |
| `RATTLER_CACHE_DIR` | `~/.cache/rattler` |

PATH additions: `~/.pixi/bin`, `~/.pixi/envs/default/bin`

## Usage

```yaml
# images.yml
my-image:
  layers:
    - pixi
```

## Used In Images

- `/overthink-images:fedora-builder` (direct)
- Transitive dependency via `python` / `supervisord` in most service images

## Related Layers

- `/overthink-layers:python` -- depends on pixi for Python 3.13 installation
- `/overthink-layers:pre-commit` -- uses pixi for pre-commit installation

## When to Use This Skill

Use when the user asks about:

- Pixi package manager setup
- Conda-forge package installation
- Python environment base layer
- The `pixi` layer or `PIXI_CACHE_DIR`
