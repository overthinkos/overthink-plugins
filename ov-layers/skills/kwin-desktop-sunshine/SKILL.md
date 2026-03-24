---
name: kwin-desktop-sunshine
description: |
  KWin desktop with Sunshine game streaming via XDG Portal capture.
  Currently disabled — KWin virtual backend lacks screencast protocol.
---

# kwin-desktop-sunshine -- KWin desktop + portal-based Sunshine

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `kwin-desktop`, `sunshine-kwin` |
| Install files | none (pure composition) |

## Usage

```yaml
# images.yml (currently disabled)
sunshine-desktop-kwin:
  enabled: false
  base: nvidia
  layers:
    - kwin-desktop-sunshine
  ports:
    - "9222:9222"
    - "47990:47990"
    - "47998:47998/udp"
    - "47999:47999/udp"
    - "48000:48000/udp"
```

## Known Limitation

KWin's `--virtual` backend does not expose `zkde_screencast_unstable_v1` Wayland protocol. Portal ScreenCast/RemoteDesktop sessions fail with response code 2. Image is disabled until KWin upstream adds screencast support for virtual backends.

## Comparison

| Stack | Capture | Input | Security | Status |
|-------|---------|-------|----------|--------|
| mutter-desktop-sunshine | Portal + PipeWire | uinput | /dev/uinput | **Working** |
| sway-desktop-sunshine | wlr-screencopy | fake-udev + uinput | /dev/uinput, NET_ADMIN | Working |
| x11-desktop-sunshine | X11 native | uinput | /dev/uinput | Working |
| niri-desktop-sunshine | wlr-screencopy (pending) | fake-udev | /dev/uinput, NET_ADMIN | Capture broken |
| **kwin-desktop-sunshine** | Portal + PipeWire | uinput | /dev/uinput | **Blocked (KWin limitation)** |

## Related Layers

- `/ov-layers:kwin-desktop` -- base desktop (composed)
- `/ov-layers:sunshine-kwin` -- Sunshine portal capture (included)
- `/ov-layers:sway-desktop-sunshine` -- Sway variant (working)
- `/ov-layers:niri-desktop-sunshine` -- Niri variant (capture broken)

## When to Use This Skill

Use when working with KWin + Sunshine streaming or comparing compositor streaming architectures.
