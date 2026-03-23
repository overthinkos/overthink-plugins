---
name: sunshine-niri
description: |
  Sunshine game streaming server configured for Niri compositor with fake-udev input injection.
  Use when working with Sunshine streaming on Niri desktop containers.
---

# sunshine-niri -- Sunshine streaming for Niri

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord`, `niri`, `pipewire` |
| Ports | `tcp:47984`, `47989`, `47990`, `tcp:48010`, `udp:47998`, `udp:47999`, `udp:48000` |
| Service | `fake-udev` (priority 15), `sunshine` (priority 20) |
| Security | `devices: [/dev/uinput]`, `cap_add: [NET_ADMIN]`, `group_add: [keep-groups]` |
| Volume | `sunshine-config` -> `~/.config/sunshine` |
| Install files | `user.yml`, `sunshine-niri-wrapper`, `fake-udev` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `LIBSEAT_BACKEND` | `noop` |

## Differences from sunshine (Sway)

| Aspect | sunshine (Sway) | sunshine-niri (Niri) |
|--------|----------------|---------------------|
| Compositor dep | `sway` | `niri` |
| WAYLAND_DISPLAY | `wayland-0` | `wayland-1` |
| Wrapper | `sunshine-wrapper` (2-phase wait) | `sunshine-niri-wrapper` (single wait) |
| Capture config | `capture = wlroots` | `capture = wlroots` |
| Sway IPC wait | Yes (30s + 2s margin) | No (niri has no sway IPC) |

The `sunshine-niri-wrapper` is simpler than `sunshine-wrapper` — only one wait phase (Wayland socket) instead of two (Wayland + sway IPC socket).

## Known Limitation

Sunshine `capture = wlroots` requires the `wlr-screencopy` Wayland protocol, which niri (Smithay) does not implement. The capture path does not work yet. See `/ov-layers:niri` for details.

## Ports

Same as `/ov-layers:sunshine` — all Sunshine/Moonlight protocol ports.

## Used In Images

Part of `/ov-layers:niri-desktop-sunshine` composition. Image: `/ov-images:sunshine-desktop-niri`.

## Related Layers

- `/ov-layers:sunshine` -- Sway variant
- `/ov-layers:niri` -- compositor dependency
- `/ov-layers:pipewire` -- audio dependency
- `/ov-layers:niri-desktop-sunshine` -- composition that includes this layer
- `/ov:sun` -- Sunshine management commands
- `/ov:moon` -- Moonlight client protocol

## When to Use This Skill

Use when working with Sunshine streaming on Niri desktop containers.
