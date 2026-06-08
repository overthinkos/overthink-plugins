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
| Layers | agent-forwarding, comfyui, dbus, charly |
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
charly box build comfyui
charly config comfyui
charly start comfyui
# Open http://localhost:8188
```

## Key Layers

- `/charly-comfyui:comfyui` — ComfyUI installation, supervisord service, volume
- `/charly-distros:nvidia` — GPU runtime and CDI device auto-detection (base)
- `/charly-distros:cuda` — CUDA toolkit and libraries (via nvidia base)
- `/charly-infrastructure:dbus-layer` — session bus for desktop notifications
- `/charly-tools:charly` — in-container `charly` binary (enables `charly eval dbus notify`)
- `/charly-distros:agent-forwarding` — SSH/GPG/direnv agent forwarding

## Related Images

- `/charly-distros:nvidia` — parent (GPU without ComfyUI)
- **CachyOS variant** — `cachyos.comfyui` is the CachyOS GPU sibling (built on the `cachyos.nvidia` GPU base) in the `overthinkos/cachyos` submodule. See `/charly-distros:cachyos`.

## Verification

After `charly start`:
- `charly status comfyui` — container running
- `charly service status comfyui` — all services RUNNING
- `curl -s -o /dev/null -w '%{http_code}' http://localhost:8188` — ComfyUI HTTP returns 200

## When to Use This Skill

**MUST be invoked** when the task involves the comfyui image, image generation, or ComfyUI workflows. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/charly-image:image` — image family umbrella (`image:` entries in `charly.yml`, build/validate/inspect/list)
- `/charly-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
