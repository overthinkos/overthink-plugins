---
name: pipewire
description: |
  PipeWire audio and media server with WirePlumber session manager.
  Use when working with audio in containers, PipeWire, or PulseAudio compatibility.
---

# pipewire -- Audio/media server for containers

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord` |
| Service | `pipewire` (supervisord, priority 5) |
| Install files | `user.yml`, `pipewire-wrapper` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `XDG_RUNTIME_DIR` | `/tmp` |

## Packages

- `pipewire` (RPM)
- `pipewire-pulseaudio` (RPM) -- PulseAudio compatibility
- `wireplumber` (RPM) -- session manager

## Usage

```yaml
# images.yml -- typically included via sway-desktop composition
my-desktop:
  layers:
    - pipewire
```

## Used In Images

Part of `/overthink-layers:sway-desktop` composition.

## Related Layers

- `/overthink-layers:supervisord` -- process manager dependency
- `/overthink-layers:sway-desktop` -- composition that includes pipewire
- `/overthink-layers:wayvnc` -- VNC server (sibling in desktop stack)

## When to Use This Skill

Use when the user asks about:

- Audio in containers
- PipeWire or PulseAudio setup
- WirePlumber session management
- Desktop audio configuration
