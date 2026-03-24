---
name: sunshine-desktop-mutter
description: |
  GNOME Mutter desktop with Sunshine portal-based streaming. First working portal-native
  headless streaming image — reduced security (no fake-udev/NET_ADMIN), NVENC encoding, AT-SPI2 auto-accept.
  MUST be invoked before building, deploying, configuring, or troubleshooting the sunshine-desktop-mutter image.
---

# sunshine-desktop-mutter

GNOME Mutter-based Sunshine streaming desktop using XDG Portal for screen capture (PipeWire) and input injection (libei/EIS). First portal-native headless streaming image that works with zero kernel-level security requirements.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia (fedora + CUDA) |
| Layers | mutter-desktop-sunshine |
| Platforms | linux/amd64 |
| Ports | 9222, 47984, 47989, 47990, 48010, 47998/udp, 47999/udp, 48000/udp |
| Registry | ghcr.io/overthinkos |

## Quick Start

```bash
ov build sunshine-desktop-mutter
ov start sunshine-desktop-mutter
# Wait ~30s for portal auto-accept to complete
ov sun passwd sunshine-desktop-mutter --generate
ov moon pair sunshine-desktop-mutter --auto
ov moon launch sunshine-desktop-mutter Desktop
# Connect with Moonlight client
```

## Architecture: Portal-Native Streaming

```
Mutter (headless) → org.gnome.Mutter.ScreenCast (D-Bus)
  ↓
xdg-desktop-portal-gnome → portal ScreenCast/RemoteDesktop
  ↓ (portal-auto-accept clicks "Share" via AT-SPI2)
PipeWire DMA-BUF stream → Sunshine (capture=portal) → NVENC → Moonlight
```

## Portal Auto-Accept

The `portal-auto-accept` supervisord service (Python, AT-SPI2) automatically accepts the GNOME screen-sharing permission dialog:
1. Monitors for `xdg-desktop-portal-gnome` dialog window
2. Toggles "Allow Remote Interaction" checkbox
3. Clicks "Share" button
4. Exits after acceptance (~4s from dialog appearance)

## Security Comparison

| | This image (Mutter) | sway-browser-sunshine (Sway) |
|---|---|---|
| `/dev/uinput` | Required | Required |
| `/dev/input` mount | Required | Required |
| `NET_ADMIN` cap | Not needed | Required |
| fake-udev daemon | Not needed | Required |
| `LIBSEAT_BACKEND` | Not needed | Required |
| `tmpfs:/run/udev` | Not needed | Required |

**Note:** `/dev/uinput` is needed because Sunshine's input subsystem uses uinput regardless of capture method. Portal RemoteDesktop/libei input is not yet implemented in Sunshine beta. When Sunshine adds libei support, uinput can be removed.

## Key Layers

- `/ov-layers:mutter-desktop-sunshine` -- full desktop composition
- `/ov-layers:mutter` -- Mutter compositor (headless, --virtual-monitor)
- `/ov-layers:sunshine-mutter` -- Sunshine with portal capture + AT-SPI2 auto-accept
- `/ov-layers:xdg-portal-gnome` -- GNOME portal backend
- `/ov-layers:cuda` -- GPU support (via nvidia base)

## Related Images

- `/ov-images:sway-browser-sunshine` -- Sway variant (working, traditional capture)
- `/ov-images:sunshine-desktop-x11` -- X11 variant (working, recommended for stability)
- `/ov-images:sunshine-desktop-kwin` -- KWin variant (disabled, protocol limitation)
- `/ov-images:sunshine-desktop-niri` -- Niri variant (capture broken)

## When to Use This Skill

Use when working with Mutter-based Sunshine streaming, portal-native capture, comparing compositor architectures, or troubleshooting GNOME portal permissions.
