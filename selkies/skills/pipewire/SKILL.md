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
| Install files | `task:`, `pipewire-wrapper` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `XDG_RUNTIME_DIR` | `/tmp` |

## Packages

- `pipewire` (RPM)
- `pipewire-pulseaudio` (RPM) -- PulseAudio compatibility
- `pipewire-utils` (RPM) -- CLI utilities (pw-cli, pw-dump, pw-top, pw-cat, pw-record, pw-play)
- `wireplumber` (RPM) -- session manager

## Usage

```yaml
# charly.yml -- typically included via sway-desktop composition
my-desktop:
  candy:
    - pipewire
```

## Used In Images

Part of `/charly-selkies:sway-desktop` composition.

## Related Layers

- `/charly-infrastructure:supervisord` -- process manager dependency
- `/charly-selkies:sway-desktop` -- composition that includes pipewire
- `/charly-selkies:wayvnc` -- VNC server (sibling in desktop stack)

## When to Use This Skill

Use when the user asks about:

- Audio in containers
- PipeWire or PulseAudio setup
- WirePlumber session management
- Desktop audio configuration

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
