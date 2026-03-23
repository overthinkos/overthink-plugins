---
name: x11-desktop-sunshine
description: |
  X11 desktop with Sunshine game streaming. All features work: capture, audio, input.
  Recommended streaming composition — no fake-udev, no Wayland hacks.
---

# x11-desktop-sunshine -- X11 desktop with Sunshine streaming

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `x11-desktop`, `sunshine-x11` |

## Usage

```yaml
# images.yml
sunshine-desktop-x11:
  base: nvidia
  layers:
    - x11-desktop-sunshine
  ports:
    - "9222:9222"
    - "47984:47984"
    - "47989:47989"
    - "47990:47990"
    - "48010:48010"
    - "47998:47998/udp"
    - "47999:47999/udp"
    - "48000:48000/udp"
```

## Comparison with Wayland alternatives

| Feature | x11-desktop-sunshine | sway-desktop-sunshine | niri-desktop-sunshine |
|---------|---------------------|----------------------|----------------------|
| Screen capture | X11 native | wlr-screencopy | Broken |
| Input (mouse/kb) | XTest native | fake-udev (erratic) | fake-udev (untested) |
| Encoders | NVENC H.264/HEVC/AV1 | NVENC | NVENC |
| fake-udev | Not needed | Required (287 lines) | Required |
| NET_ADMIN cap | Not needed | Required | Required |
| Workarounds | 0 | 12+ | 6+ |

## Used In Images

- `/ov-images:sunshine-desktop-x11`

## Related Layers

- `/ov-layers:sway-desktop-sunshine` -- Sway variant (capture works, input broken)
- `/ov-layers:niri-desktop-sunshine` -- Niri variant (capture broken)
- `/ov-layers:x11-desktop` -- base desktop (composed)
- `/ov-layers:sunshine-x11` -- Sunshine X11 (included)
- `/ov:sun` -- Sunshine management commands
- `/ov:moon` -- Moonlight client protocol

## When to Use This Skill

Use when working with Sunshine streaming desktop containers. This is the recommended composition for working streaming.
