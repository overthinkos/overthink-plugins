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
| Install files | `layer.yml`, `tasks:` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `PIXI_CACHE_DIR` | `~/.cache/pixi` |
| `RATTLER_CACHE_DIR` | `~/.cache/rattler` |

PATH additions: `~/.pixi/bin`, `~/.pixi/envs/default/bin`

## Usage

```yaml
# image.yml
my-image:
  layers:
    - pixi
```

## Used In Images

- `/ov-distros:fedora-builder` (direct)
- `/ov-distros:archlinux-builder` (direct)
- Transitive dependency via `python` / `supervisord` in most service images

## Related Layers

- `/ov-languages:python` -- depends on pixi for Python 3.13 installation
- `/ov-coder:pre-commit` -- uses pixi for pre-commit installation

## When to Use This Skill

Use when the user asks about:

- Pixi package manager setup
- Conda-forge package installation
- Python environment base layer
- The `pixi` layer or `PIXI_CACHE_DIR`

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
