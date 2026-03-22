---
name: steam
description: |
  Steam gaming client with gamescope, XWayland, and Sunshine Big Picture app.
  Use when working with Steam, gaming, gamescope, or XWayland in containers.
---

# steam -- Steam gaming client

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `sway` |
| Security | `shm_size: 1g` |
| Volume | `steam-data` -> `~/.local/share/Steam` |
| Install files | `user.yml` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `STEAM_RUNTIME_PREFER_HOST_LIBRARIES` | `0` |

## Packages

- `steam` (RPM, from RPM Fusion nonfree)
- `gamescope` (RPM, Fedora main repos)
- `xorg-x11-server-Xwayland` (RPM)

## Usage

```yaml
# images.yml — requires nvidia base (RPM Fusion nonfree via fedora-nonfree)
sway-browser-sunshine-steam:
  base: nvidia
  layers:
    - sway-desktop-sunshine
    - steam
```

## XWayland

Steam is an X11 application — it requires XWayland. The base sway config has `xwayland disable` for headless optimization. This layer installs a sway drop-in config at `~/.config/sway/config.d/xwayland.conf` that enables XWayland.

## Sunshine Apps

The `user.yml` pre-configures `~/.config/sunshine/apps.json` with:
- **Desktop** — default full desktop session
- **Steam Big Picture** — launches via `setsid steam steam://open/bigpicture` (detached pattern)

Steam's self-updater kills the original process, so the `detached` command pattern with `setsid` is required. The `undo` command (`steam steam://close/bigpicture`) closes Big Picture when the Moonlight streaming session ends.

## Gamescope

Gamescope is a nested Wayland compositor for games. Use it in Steam launch options:

```
gamescope -W 1920 -H 1080 -r 60 -- %command%
```

Features: resolution spoofing, FSR/NIS upscaling, FPS limiting, HDR support.

## First Login

Steam Guard requires interactive login. Connect via Moonlight "Desktop" app, launch Steam, and log in manually. Auth tokens persist in the `steam-data` volume for subsequent container restarts.

## Related Layers

- `/ov-layers:sway` — Wayland compositor (dependency)
- `/ov-layers:sunshine` — Game streaming server
- `/ov-layers:sway-desktop-sunshine` — Desktop composition with Sunshine
- `/ov-layers:cuda` — NVIDIA GPU support (via nvidia base)

## When to Use This Skill

Use when the user asks about:

- Steam installation in containers
- Gaming on Sway/Wayland containers
- Gamescope configuration
- XWayland setup
- Steam Big Picture via Sunshine/Moonlight
