---
name: comfyui
description: |
  ComfyUI image generation server with CUDA GPU support.
  Runs as a supervisord service on port 8188 with persistent storage.
  MUST be invoked before building, deploying, configuring, or troubleshooting the comfyui image.
---

# comfyui

GPU-accelerated ComfyUI image generation server with node-based workflow UI.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia |
| Layers | agent-forwarding, comfyui, dbus, ov |
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
ov image build comfyui
ov config comfyui
ov start comfyui
# Open http://localhost:8188
```

## Key Layers

- `/ov-layers:comfyui` — ComfyUI installation, supervisord service, volume
- `/ov-layers:nvidia` — GPU runtime and CDI device auto-detection (base)
- `/ov-layers:cuda` — CUDA toolkit and libraries (via nvidia base)
- `/ov-layers:dbus` — session bus for desktop notifications
- `/ov-layers:ov` — in-container `ov` binary (enables `ov test dbus notify`)
- `/ov-layers:agent-forwarding` — SSH/GPG/direnv agent forwarding

## Related Images

- `/ov-images:nvidia` — parent (GPU without ComfyUI)

## Verification

After `ov start`:
- `ov status comfyui` — container running
- `ov service status comfyui` — all services RUNNING
- `curl -s -o /dev/null -w '%{http_code}' http://localhost:8188` — ComfyUI HTTP returns 200

## When to Use This Skill

**MUST be invoked** when the task involves the comfyui image, image generation, or ComfyUI workflows. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov:build` — `build.yml` vocabulary (distros, builders, init-systems)
