---
name: steam
description: |
  Steam gaming client with gamescope.
  Use when working with Steam, gaming, or gamescope in containers.
---

# steam -- Steam gaming client

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `sway` |
| Security | `shm_size: 1g` |
| Volume | `steam-data` -> `~/.local/share/Steam` |
| Install files | `tasks:` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `STEAM_RUNTIME_PREFER_HOST_LIBRARIES` | `0` |

## Packages

- `steam` (RPM, from RPM Fusion nonfree)
- `gamescope` (RPM, Fedora main repos)

XWayland is provided by the base `sway` layer.

## Usage

```yaml
# image.yml — requires nvidia base (RPM Fusion nonfree via fedora-nonfree)
sway-browser-vnc-steam:
  base: nvidia
  layers:
    - sway-desktop-vnc
    - steam
```

## XWayland

Steam is an X11 application — it requires XWayland. The base sway config has `xwayland disable` for headless optimization. This layer installs a sway drop-in config at `~/.config/sway/config.d/xwayland.conf` that enables XWayland.

## Gamescope

Gamescope is a nested Wayland compositor for games. Use it in Steam launch options:

```
gamescope -W 1920 -H 1080 -r 60 -- %command%
```

Features: resolution spoofing, FSR/NIS upscaling, FPS limiting, HDR support.

## First Login

Steam Guard requires interactive login. Connect via VNC desktop, launch Steam, and log in manually. Auth tokens persist in the `steam-data` volume for subsequent container restarts.

## Related Layers

- `/ov-layers:sway` — Wayland compositor (dependency)
- `/ov-layers:sway-desktop-vnc` — Desktop composition with VNC
- `/ov-layers:cuda` — NVIDIA GPU support (via nvidia base)

## Used In Images

Not used in any current image definition. Standalone gaming layer requiring a Sway desktop image.

## When to Use This Skill

Use when the user asks about:

- Steam installation in containers
- Gaming on Sway/Wayland containers
- Gamescope configuration
- XWayland setup

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
