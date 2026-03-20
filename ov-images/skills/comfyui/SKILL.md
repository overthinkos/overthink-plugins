---
name: comfyui
description: |
  ComfyUI image generation server with CUDA GPU support.
  Runs as a supervisord service on port 8188 with persistent storage.
  Use when working with ComfyUI, image generation, or Stable Diffusion.
---

# comfyui

GPU-accelerated ComfyUI image generation server with node-based workflow UI.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia |
| Layers | comfyui |
| Platforms | linux/amd64 |
| Ports | 8188 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` → `nvidia` (CUDA base)
2. `pixi` → `python` → `supervisord` (transitive)
3. `comfyui` — ComfyUI server, models/output volume

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 8188 | ComfyUI web UI | HTTP |

## Quick Start

```bash
ov build comfyui
ov enable comfyui
ov start comfyui
# Open http://localhost:8188
```

## Key Layers

- `/ov-layers:comfyui` — ComfyUI installation, supervisord service, volume
- `/ov-layers:cuda` — GPU support (via nvidia base)

## Related Images

- `/ov-images:nvidia` — parent (GPU without ComfyUI)

## Verification

After `ov start`:
- `ov status comfyui` — container running
- `ov service status comfyui` — all supervisord services RUNNING
- `curl -s -o /dev/null -w '%{http_code}' http://localhost:8188` — ComfyUI HTTP returns 200

## When to Use This Skill

Use when the user asks about the comfyui image, image generation, Stable Diffusion, or ComfyUI workflows.
