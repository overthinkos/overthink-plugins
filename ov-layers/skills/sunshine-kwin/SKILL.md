---
name: sunshine-kwin
description: |
  Sunshine game streaming with XDG Portal capture for KWin (no uinput/fake-udev).
  Use when working with portal-based Sunshine streaming on KWin.
---

# sunshine-kwin -- Sunshine with portal capture for KWin

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord`, `kwin`, `pipewire` |
| Ports | `tcp:47984`, `47989`, `47990`, `tcp:48010`, `udp:47998`, `udp:47999`, `udp:48000` |
| Service | `sunshine` (supervisord, priority 20) |
| Volume | `sunshine-config` -> `~/.config/sunshine` |
| Install files | `layer.yml`, `user.yml`, `sunshine-kwin-wrapper` |

## Packages

- `sunshine` (RPM, from COPR `lizardbyte/beta`)

## Security: Reduced vs Sway

Would require `/dev/uinput` and `/dev/input` mount for input if ever enabled (Sunshine's input subsystem uses uinput regardless of capture method). Eliminates fake-udev, NET_ADMIN, LIBSEAT_BACKEND vs Sway.

| | sunshine (Sway) | sunshine-kwin |
|---|---|---|
| `/dev/uinput` | Required | Would be required |
| `/dev/input` mount | Required | Would be required |
| `tmpfs:/run/udev` | Required | Not needed |
| `NET_ADMIN` cap | Required | Not needed |
| fake-udev service | Required | Not needed |

**Note:** Currently no security section declared because the image is disabled (KWin screencast limitation). If enabled, `/dev/uinput` + `/dev/input` mount would need to be added for input to work.

## Sunshine Configuration

The wrapper generates `sunshine.conf` with `capture = portal` (not `wlroots` or `x11`).

## Known Limitation

KWin's `--virtual` backend does not expose `zkde_screencast_unstable_v1`. Portal ScreenCast/RemoteDesktop CreateSession fails with response code 2. **Streaming does not work** as of KDE 6.6.3. The image is disabled (`enabled: false`).

For working Sunshine streaming, use `/ov-layers:sunshine` (Sway) or `/ov-layers:sunshine-x11` (X11).

## Related Layers

- `/ov-layers:kwin` -- compositor dependency
- `/ov-layers:sunshine` -- Sway variant (working, uses fake-udev)
- `/ov-layers:sunshine-x11` -- X11 variant (working, uses XTest)
- `/ov-layers:sunshine-niri` -- Niri variant (capture broken)
- `/ov-layers:kwin-desktop-sunshine` -- composition that includes this layer

## When to Use This Skill

Use when working with:

- Portal-based Sunshine streaming on KWin
- Comparing Sunshine capture methods (portal vs wlroots vs x11)
- KWin screencast/RemoteDesktop portal limitations
