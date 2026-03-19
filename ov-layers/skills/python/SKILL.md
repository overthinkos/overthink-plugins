---
name: python
description: |
  Python 3.13 runtime installed via pixi (conda-forge).
  Use when working with Python, pixi environments, or Python dependencies.
---

# python -- Python 3.13 via pixi

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `pixi` |
| Install files | `layer.yml`, `pixi.toml` |

## Packages

Pixi (conda-forge): `python >=3.13,<3.14`

## Usage

```yaml
# images.yml
my-image:
  layers:
    - python
```

## Used In Images

- Transitive dependency via `supervisord` in most service images (`openclaw`, `jupyter`, `ollama`, `comfyui`, `immich`, etc.)

## Related Layers

- `/ov-layers:pixi` -- required dependency (provides pixi binary)
- `/ov-layers:python-ml` -- ML-focused Python with CUDA support
- `/ov-layers:language-runtimes` -- depends on python

## When to Use This Skill

Use when the user asks about:

- Python runtime in containers
- Pixi-managed Python installation
- The `python` layer
- Why `pip install` and `conda install` are not used (pixi is the only Python package manager)
