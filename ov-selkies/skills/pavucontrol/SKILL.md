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

- `/ov-selkies:sway-desktop` -- via metalayer composition
- `/ov-selkies:selkies-desktop` -- via metalayer composition

## Used In Images

- `/ov-selkies:sway-browser-vnc` (via `sway-desktop` metalayer)
- `/ov-openclaw:openclaw-sway-browser` (via `sway-desktop` metalayer)
- `/ov-openclaw:openclaw-ollama-sway-browser` (via `sway-desktop` metalayer)
- `/ov-selkies:selkies-desktop` (via `selkies-desktop` metalayer)
- `/ov-selkies:selkies-desktop-nvidia` (via `selkies-desktop` metalayer)

## Related Layers

- `/ov-selkies:pipewire` -- audio server (provides PulseAudio compatibility)
- `/ov-selkies:waybar` -- volume indicator with pavucontrol on-click
- `/ov-selkies:waybar-labwc` -- same volume indicator

## When to Use This Skill

Use when the user asks about:

- Audio volume control in desktop containers
- PulseAudio/PipeWire GUI configuration
- pavucontrol setup

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
