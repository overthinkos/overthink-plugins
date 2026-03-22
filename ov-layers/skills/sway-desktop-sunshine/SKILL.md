---
name: sway-desktop-sunshine
description: |
  GPU-accelerated Sway desktop with Sunshine game streaming instead of VNC.
  Use when working with Sunshine-based desktop containers or GPU-optimized remote desktop.
---

# sway-desktop-sunshine -- GPU-accelerated remote desktop composition

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `sway-desktop`, `sunshine` |
| Install files | none (pure composition) |

## Usage

```yaml
# images.yml
sway-browser-sunshine:
  base: nvidia
  layers:
    - sway-desktop-sunshine
  ports:
    - "9222:9222"
    - "47990:47990"
    - "47998:47998/udp"
    - "47999:47999/udp"
    - "48000:48000/udp"
```

## Comparison with sway-desktop

Composes `sway-desktop` (base) with `sunshine` (display server):

| Component | sway-desktop-vnc | sway-desktop-sunshine |
|-----------|-----------------|----------------------|
| Remote access | wayvnc (VNC, port 5900) | Sunshine (Moonlight, port 47990) |
| NVIDIA renderer | gles2 (GPU) | gles2 (GPU) |
| Video encoding | N/A (raw VNC framebuffer) | NVENC H.264/HEVC/AV1 |
| Screenshots | `ov wl screenshot` (grim) | `ov wl screenshot` (grim) |

Both use the same gles2 renderer on NVIDIA. The difference is the streaming protocol (VNC vs Moonlight/GameStream).

## Used In Images

- `/ov-images:sway-browser-sunshine`
- `/ov-images:sway-browser-sunshine-steam` (+ steam layer)
- `/ov-images:sway-browser-sunshine-steam-heroic` (+ steam + heroic)
- `/ov-images:openclaw-ollama-sway-sunshine`

## Related Layers

- `/ov:sun` -- Sunshine management commands (credentials, pairing, config)
- `/ov-layers:sway-desktop` -- base desktop (composed)
- `/ov-layers:sway-desktop-vnc` -- VNC-based variant
- `/ov-layers:sunshine` -- Sunshine game streaming (included)
- `/ov-layers:pipewire` -- audio/media server (included)
- `/ov-layers:chrome-sway` -- Chrome browser on Sway (included)
- `/ov-layers:waybar` -- status bar and auto-tiling (included)

## When to Use This Skill

Use when the user asks about:

- GPU-accelerated desktop containers
- Sunshine-based remote desktop composition
- Comparing sway-desktop vs sway-desktop-sunshine
- NVIDIA-optimized container desktops
