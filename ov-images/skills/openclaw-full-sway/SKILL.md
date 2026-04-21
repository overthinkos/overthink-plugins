---
name: openclaw-full-sway
description: |
  OpenClaw with all tools + Sway desktop + VNC. No GPU/ML.
  Currently disabled. Enable in image.yml to build.
  MUST be invoked before building, deploying, or troubleshooting the openclaw-full-sway image.
---

# openclaw-full-sway

OpenClaw with maximal tool coverage plus Sway desktop and VNC access.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, openclaw-full, sway-desktop-vnc |
| Platforms | linux/amd64 |
| Ports | 18789 (gateway), 5900 (VNC), 9222 (CDP) |
| Status | **disabled** (set `enabled: true` in image.yml) |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (external base)
2. `openclaw-full` metalayer (28 tool layers)
3. `sway-desktop-vnc` metalayer:
   - `sway-desktop` (pipewire + chrome-sway + terminal + waybar)
   - `wayvnc` — VNC server on :5900

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway | HTTP |
| 5900 | VNC | TCP |
| 9222 | Chrome DevTools | HTTP |

## Quick Start

```bash
# Enable in image.yml first (remove enabled: false)
ov image build openclaw-full-sway
ov config openclaw-full-sway
ov start openclaw-full-sway
```

## Key Layers

- `/ov-layers:openclaw-full` — All tool layers
- `/ov-layers:sway-desktop-vnc` — Desktop with VNC access

## Related Images

- `/ov-images:openclaw-sway-browser` — enabled variant (same concept, uses openclaw-full metalayer)
- `/ov-images:openclaw-full` — headless variant (no desktop)
- `/ov-images:openclaw-full-ml` — with ML/Ollama (disabled)

## When to Use This Skill

**MUST be invoked** when the task involves the openclaw-full-sway image.

## Related

- `/ov:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov:build` — `build.yml` vocabulary (distros, builders, init-systems)
