---
name: niri
description: |
  Niri Wayland compositor (Smithay-based) built from source with virtual output support.
  Use when working with niri compositor, headless Wayland, or Smithay-based desktop containers.
---

# niri -- Headless Smithay Wayland compositor

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `dbus`, `rust`, `build-toolchain` |
| Service | `niri` (supervisord, priority 10) |
| Install files | `tasks:`, `config.kdl`, `niri-wrapper` |
| Built from | `QaidVoid/niri` `feat/virtual` branch (Smithay + virtual outputs) |

## Environment Variables

| Variable | Value |
|----------|-------|
| `WAYLAND_DISPLAY` | `wayland-1` |
| `XDG_RUNTIME_DIR` | `/tmp` |
| `XDG_CURRENT_DESKTOP` | `niri` |
| `EGL_LOG_LEVEL` | `fatal` |

**Note:** Niri/Smithay creates `wayland-1` (not `wayland-0` like wlroots). All downstream services must use `wayland-1`.

## Build From Source

Niri is built from the QaidVoid/niri fork (`feat/virtual` branch) which adds:
- `NIRI_BACKEND=headless` mode — creates virtual outputs without DRM/KMS
- `niri msg create-virtual-output` — dynamic virtual output creation
- Timer-based vblank simulation for proper frame pacing

Build process (in `tasks:`):
1. `git clone --depth 1 --branch feat/virtual https://github.com/QaidVoid/niri.git`
2. `cargo build --release`
3. Install binary to `/usr/local/bin/niri`
4. Clean up build deps to reduce image size

## Headless Mode

The `niri-wrapper` starts niri with `NIRI_BACKEND=headless`:
- Creates `HEADLESS-1` virtual output automatically (1920x1080@60Hz)
- No GPU-specific env vars needed (unlike sway which needs 4 NVIDIA vars + `--unsupported-gpu`)
- No `cap_sys_nice` stripping needed (niri is a Rust binary, no file capabilities)
- No `WLR_BACKENDS` coordination needed

## Differences from Sway

| Aspect | Sway (wlroots) | Niri (Smithay) |
|--------|----------------|----------------|
| Headless mode | `WLR_BACKENDS=headless` | `NIRI_BACKEND=headless` |
| Socket name | `wayland-0` | `wayland-1` |
| GPU workarounds | 4 env vars + `--unsupported-gpu` | None needed |
| File capabilities | `cap_sys_nice` must be stripped | No file caps |
| IPC | sway-ipc socket | niri IPC socket |
| Config format | i3-style | KDL |
| Screen capture | wlr-screencopy | ext-image-copy-capture (pending) |

## Known Limitation: Screen Capture

Niri does not implement the `wlr-screencopy` protocol (it's wlroots-specific). The `ext-image-copy-capture` protocol support is pending.

## Used In Images

Part of `niri-desktop` composition.

## Related Layers

- `/ov-layers:sway` -- wlroots compositor alternative
- `/ov-layers:dbus` -- D-Bus session bus dependency
- `/ov-layers:chrome-niri` -- Chrome browser on Niri
- `/ov-layers:niri-apps` -- Desktop apps (terminal, file manager)
- `/ov-layers:niri-desktop` -- Full desktop composition

## When to Use This Skill

Use when the user asks about:

- Niri compositor configuration
- Smithay-based Wayland setup
- Headless virtual output support
- Alternative to Sway for containers
- Building niri from source

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
