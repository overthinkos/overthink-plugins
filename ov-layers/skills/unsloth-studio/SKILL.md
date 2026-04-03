---
name: unsloth-studio
description: |
  Unsloth Studio web UI metalayer composing unsloth + supervisord service on ports 8888 and 8000.
  Use when working with Unsloth Studio, the fine-tuning web UI, or the unsloth-studio image.
---

# unsloth-studio -- Unsloth Studio web UI service

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord` |
| Composed layers | `unsloth` |
| Ports | 8888 (Studio UI), 8000 (vLLM API) |
| Volumes | `workspace` -> `~/workspace` |
| Service | `unsloth-studio` (supervisord) |

## Service

Runs `pixi run start-studio` which executes `unsloth studio -H 0.0.0.0 -p 8888`. The Studio launches its own vLLM API server on port 8000 for inference and synthetic data generation.

## Usage

```yaml
# images.yml
unsloth-studio:
  base: nvidia
  layers:
    - unsloth-studio
  ports:
    - "8888:8888"
    - "8000:8000"
```

## Used In Images

- `/ov-images:unsloth-studio`

## Related Layers

- `/ov-layers:unsloth` -- Core ML environment (composed into this metalayer)
- `/ov-layers:supervisord` -- Process manager dependency
- `/ov-layers:jupyter` -- Alternative ML UI (Jupyter notebooks instead of Studio)

## When to Use This Skill

Use when the user asks about:

- Unsloth Studio web UI setup
- The unsloth-studio service or container
- Port 8888 or 8000 in the context of fine-tuning
- Training dashboard or Data Recipes UI
