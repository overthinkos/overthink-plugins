---
name: sunshine-desktop-kwin
description: |
  KWin desktop with Sunshine portal-based streaming. Currently disabled.
  KWin virtual backend lacks zkde_screencast protocol needed for portal capture.
  MUST be invoked before building, deploying, or troubleshooting the sunshine-desktop-kwin image.
---

# sunshine-desktop-kwin

KWin-based Sunshine streaming desktop using XDG Portal for capture and libei/EIS for input. Currently **disabled** -- KWin's `--virtual` backend does not expose `zkde_screencast_unstable_v1`, blocking portal ScreenCast/RemoteDesktop sessions.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia (fedora + CUDA) |
| Layers | kwin-desktop-sunshine |
| Platforms | linux/amd64 |
| Ports | 9222, 47984, 47989, 47990, 48010, 47998/udp, 47999/udp, 48000/udp |
| Status | **Disabled** (`enabled: false`) |

## Why Disabled

KWin 6.6.3 `--virtual` backend does not register the `zkde_screencast_unstable_v1` Wayland protocol. Without it, `xdg-desktop-portal-kde` immediately rejects ScreenCast/RemoteDesktop CreateSession requests (response code 2). Confirmed on both container (KWin 6.6.3) and host (KWin 6.5.5).

The screencast plugin loads and the EIS (libei) plugin loads, but the protocol global is never advertised to Wayland clients. This is an upstream KWin limitation.

## Architecture: Portal vs Traditional

| Aspect | KWin + Portal (this image) | Sway + uinput (working) | X11 + XTest (working) |
|--------|---------------------------|------------------------|-----------------------|
| Screen capture | XDG Portal + PipeWire | wlr-screencopy | Native X11 |
| Input injection | libei/EIS (userspace) | uinput (kernel) | XTest (X11) |
| `/dev/uinput` | Not needed | Required | Required |
| `cap_sys_admin` | Not needed | Not needed | Not needed |
| `NET_ADMIN` | Not needed | Required | Not needed |
| fake-udev | Not needed | Required | Not needed |
| Status | Blocked | Working | Working |

## Key Layers

- `/ov-layers:kwin-desktop-sunshine` -- full desktop composition with Sunshine
- `/ov-layers:kwin` -- KWin Wayland compositor (virtual backend)
- `/ov-layers:sunshine-kwin` -- Sunshine with portal capture (no security declarations)
- `/ov-layers:xdg-portal-kde` -- KDE portal backend
- `/ov-layers:cuda` -- GPU support (via nvidia base)

## Related Images

- `/ov-images:sway-browser-sunshine` -- Sway variant (working)
- `/ov-images:sunshine-desktop-x11` -- X11 variant (working, recommended)
- `/ov-images:sunshine-desktop-niri` -- Niri variant (capture broken)

## When to Use This Skill

Use when working with KWin-based Sunshine streaming, comparing portal-based vs traditional capture, or tracking the upstream KWin screencast limitation.
