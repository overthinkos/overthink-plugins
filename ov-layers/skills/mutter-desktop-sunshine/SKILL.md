---
name: mutter-desktop-sunshine
description: |
  GNOME Mutter desktop with Sunshine portal-based game streaming.
  First working portal-native headless streaming composition — reduced security vs Sway (no fake-udev/NET_ADMIN).
---

# mutter-desktop-sunshine -- Mutter desktop + portal-based Sunshine

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `mutter-desktop`, `sunshine-mutter` |
| Install files | none (pure composition) |

## Usage

```yaml
# images.yml
sunshine-desktop-mutter:
  base: nvidia
  layers:
    - mutter-desktop-sunshine
  ports:
    - "9222:9222"
    - "47990:47990"
    - "47998:47998/udp"
    - "47999:47999/udp"
    - "48000:48000/udp"
```

## Comparison

| Stack | Capture | Input | Security | Status |
|-------|---------|-------|----------|--------|
| **mutter-desktop-sunshine** | XDG Portal + PipeWire | uinput | /dev/uinput | **Working** |
| sway-desktop-sunshine | wlr-screencopy | fake-udev + uinput | /dev/uinput, NET_ADMIN, /run/udev | Working |
| x11-desktop-sunshine | X11 native | uinput | /dev/uinput | Working |
| kwin-desktop-sunshine | Portal (blocked) | uinput | /dev/uinput | Disabled |
| niri-desktop-sunshine | wlr-screencopy (broken) | fake-udev | /dev/uinput, NET_ADMIN | Broken |

## Related Layers

- `/ov-layers:mutter-desktop` -- base desktop (composed)
- `/ov-layers:sunshine-mutter` -- Sunshine portal capture (included)
- `/ov-layers:sway-desktop-sunshine` -- Sway variant (working)
- `/ov-layers:kwin-desktop-sunshine` -- KWin variant (disabled)

## When to Use This Skill

Use when working with Mutter + Sunshine streaming or comparing compositor streaming architectures.
