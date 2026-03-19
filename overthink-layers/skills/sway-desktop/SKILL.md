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
| Layers (composition) | `pipewire`, `wayvnc`, `chrome-sway`, `xfce4-terminal`, `thunar`, `waybar` |
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

- `/overthink-images:openclaw-sway-browser`
- `/overthink-images:openclaw-ollama-sway-browser`

## Related Layers

- `/overthink-layers:pipewire` -- audio/media server (included)
- `/overthink-layers:wayvnc` -- VNC server on tcp:5900 (included)
- `/overthink-layers:chrome-sway` -- Chrome browser on Sway (included)
- `/overthink-layers:waybar` -- status bar and auto-tiling (included)
- `/overthink-layers:sway` -- Wayland compositor (transitive via chrome-sway)
- `/overthink-layers:dbus` -- D-Bus session bus (transitive via sway)

## When to Use This Skill

Use when the user asks about:

- Full desktop container setup
- Remote desktop with VNC access
- Browser-enabled container images
- Sway desktop composition
- Desktop applications in containers
