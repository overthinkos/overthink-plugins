---
name: comfyui
description: |
  ComfyUI image generation service on port 8188 with CUDA GPU support.
  Use when working with ComfyUI, image generation, Stable Diffusion, or AI art pipelines.
---

# comfyui -- GPU-accelerated image generation service

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `cuda`, `supervisord` |
| Ports | 8188 |
| Volumes | `comfyui` -> `~/ComfyUI` |
| Service | `comfyui` (supervisord) |
| Install files | `user.yml`, `pixi.toml` |

## Packages

- `aria2` (RPM) -- download manager for model files
- `git-lfs` (RPM) -- large file support for model repos

## Environment Variables

| Variable | Value |
|----------|-------|
| (inherited from cuda) | CUDA toolkit paths |

## Usage

```yaml
# images.yml
comfyui:
  layers:
    - comfyui
```

## Used In Images

- `/overthink-images:comfyui`

## Related Layers

- `/overthink-layers:cuda` -- CUDA toolkit dependency
- `/overthink-layers:supervisord` -- process manager dependency

## When to Use This Skill

Use when the user asks about:

- ComfyUI setup or configuration
- Image generation service
- Stable Diffusion workflows
- The comfyui volume or port 8188
- GPU-accelerated AI art
