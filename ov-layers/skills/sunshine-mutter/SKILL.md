---
name: sunshine-mutter
description: |
  Sunshine game streaming with XDG Portal capture for Mutter (no uinput/fake-udev).
  First working portal-based headless streaming stack. Uses AT-SPI2 auto-accept for portal dialog.
---

# sunshine-mutter -- Sunshine with portal capture for Mutter

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord`, `mutter`, `pipewire` |
| Ports | `tcp:47984`, `47989`, `47990`, `tcp:48010`, `udp:47998`, `udp:47999`, `udp:48000` |
| Services | `sunshine` (priority 20), `portal-accept` (priority 21) |
| Volume | `sunshine-config` -> `~/.config/sunshine` |
| Install files | `layer.yml`, `user.yml`, `sunshine-mutter-wrapper`, `portal-auto-accept` |

## Packages

- `sunshine` (RPM, from COPR `lizardbyte/beta`)

## Security: Reduced vs Sway

Requires `/dev/uinput` and `/dev/input` mount for input (Sunshine's input subsystem uses uinput regardless of capture method — portal RemoteDesktop/libei is not yet implemented in Sunshine beta). But eliminates fake-udev, NET_ADMIN, and LIBSEAT_BACKEND vs the Sway stack.

| | sunshine (Sway) | sunshine-mutter |
|---|---|---|
| `/dev/uinput` | Required | Required |
| `/dev/input` mount | Required | Required |
| `tmpfs:/run/udev` | Required | Not needed |
| `NET_ADMIN` cap | Required | Not needed |
| fake-udev service | Required | Not needed |
| `LIBSEAT_BACKEND` | Required | Not needed |

Screen capture uses portal + PipeWire (no `cap_sys_admin`). Input uses uinput (same as X11 stack). When Sunshine adds portal RemoteDesktop/libei support, `/dev/uinput` and `/dev/input` can be removed.

## Portal Auto-Accept (AT-SPI2)

The `portal-auto-accept` service (priority 21) automatically clicks "Share" on the GNOME portal permission dialog using AT-SPI2 accessibility APIs:
1. Waits for `xdg-desktop-portal-gnome` dialog to appear
2. Toggles "Allow Remote Interaction" checkbox
3. Clicks "Share" button
4. Exits after acceptance

Uses `/usr/bin/python3` (system Python, not pixi) with `python3-gobject` for AT-SPI2 access.

## Sunshine Configuration

The wrapper generates `sunshine.conf` with `capture = portal` and `encoder = nvenc`.

## Related Layers

- `/ov-layers:mutter` -- compositor dependency
- `/ov-layers:sunshine` -- Sway variant (working, uses fake-udev + /dev/uinput)
- `/ov-layers:sunshine-x11` -- X11 variant (working, uses XTest)
- `/ov-layers:sunshine-kwin` -- KWin variant (disabled, protocol missing)
- `/ov-layers:mutter-desktop-sunshine` -- composition that includes this layer

## When to Use This Skill

Use when working with:

- Portal-based Sunshine streaming on Mutter/GNOME
- AT-SPI2 portal dialog automation
- Comparing portal vs traditional capture approaches
