---
name: sway-desktop
description: |
  Full Sway desktop composition with audio, VNC, Chrome browser, terminal, file manager, and status bar.
  Use when working with desktop containers, remote desktop, or browser-enabled Sway images.
---

# sway-desktop -- Full remote desktop composition

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `pipewire`, `xdg-portal`, `wl-tools`, `wayvnc`, `chrome-sway`, `xfce4-terminal`, `thunar`, `waybar` |
| Install files | none (pure composition) |

## Usage

```yaml
# images.yml
openclaw-sway-browser:
  layers:
    - sway-desktop
    - openclaw
```

## Used In Images

- `/ov-images:openclaw-sway-browser`
- `/ov-images:openclaw-ollama-sway-browser`

## GPU-Accelerated Variant

For NVIDIA-optimized images, use `/ov-layers:sway-desktop-sunshine` instead. It replaces wayvnc with Sunshine game streaming, enabling full GPU acceleration (gles2 + NVENC) on NVIDIA headless.

## Related Layers

- `/ov-layers:pipewire` -- audio/media server (included)
- `/ov-layers:xdg-portal` -- XDG desktop portal infrastructure (included)
- `/ov-layers:wl-tools` -- Wayland automation tools: grim, wtype, wlrctl (included)
- `/ov-layers:wayvnc` -- VNC server on tcp:5900 (included)
- `/ov-layers:chrome-sway` -- Chrome browser on Sway (included)
- `/ov-layers:waybar` -- status bar and auto-tiling (included)
- `/ov-layers:sway` -- Wayland compositor (transitive via chrome-sway)
- `/ov-layers:dbus` -- D-Bus session bus (transitive via sway)
- `/ov-layers:sway-desktop-sunshine` -- GPU-accelerated variant with Sunshine

## When to Use This Skill

Use when the user asks about:

- Full desktop container setup
- Remote desktop with VNC access
- Browser-enabled container images
- Sway desktop composition
- Desktop applications in containers
