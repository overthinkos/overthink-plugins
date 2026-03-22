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
| Layers (composition) | `pipewire`, `xdg-portal`, `wl-tools`, `sunshine`, `chrome-sway`, `xfce4-terminal`, `thunar`, `waybar` |
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

Identical to `sway-desktop` except `wayvnc` is replaced with `sunshine`:

| Component | sway-desktop | sway-desktop-sunshine |
|-----------|-------------|----------------------|
| Remote access | wayvnc (VNC, port 5900) | Sunshine (Moonlight, port 47990) |
| NVIDIA headless renderer | pixman (CPU) | gles2 (GPU) |
| Video encoding | Raw framebuffer | NVENC H.264/HEVC/AV1 |
| Chrome rendering | Software fallback | GPU-accelerated |

## Used In Images

- `/ov-images:sway-browser-sunshine`
- `/ov-images:sway-browser-sunshine-steam` (+ steam layer)
- `/ov-images:openclaw-ollama-sway-sunshine`

## Related Layers

- `/ov:sun` -- Sunshine management commands (credentials, pairing, config)
- `/ov-layers:sway-desktop` -- VNC-based variant
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
