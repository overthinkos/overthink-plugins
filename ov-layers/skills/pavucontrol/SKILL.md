---
name: pavucontrol
description: |
  PulseAudio volume control GUI for desktop containers with PipeWire audio.
  Use when working with audio configuration or volume control.
---

# pavucontrol -- PulseAudio Volume Control

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `pipewire` |
| Service | none (GUI app, launched on demand) |
| Install files | `layer.yml` only |

## Packages

- `pavucontrol` (RPM)

## Usage

Launched on demand from waybar's pulseaudio module (click the volume indicator) or from a terminal:

```bash
pavucontrol &
```

## Waybar Integration

The waybar `pulseaudio` module has `"on-click": "pavucontrol"` — clicking the volume indicator in the status bar opens pavucontrol.

## Used In

- `/ov-layers:sway-desktop` -- via metalayer composition
- `/ov-layers:selkies-desktop` -- via metalayer composition

## Related Layers

- `/ov-layers:pipewire` -- audio server (provides PulseAudio compatibility)
- `/ov-layers:waybar` -- volume indicator with pavucontrol on-click
- `/ov-layers:waybar-labwc` -- same volume indicator

## When to Use This Skill

Use when the user asks about:

- Audio volume control in desktop containers
- PulseAudio/PipeWire GUI configuration
- pavucontrol setup
