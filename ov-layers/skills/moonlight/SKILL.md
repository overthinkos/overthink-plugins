---
name: moonlight
description: |
  Moonlight game streaming client (AppImage from GitHub). Connects to Sunshine servers.
  Use when working with Moonlight client, game streaming testing, or Sunshine debugging.
---

# moonlight -- Moonlight Game Streaming Client

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `sway` |
| Packages | `fuse-libs` (RPM), `moonlight-qt` v6.1.0 (AppImage from GitHub) |
| Install files | `root.yml` (AppImage download), `user.yml` (sway keybinding) |

## Usage

```yaml
# images.yml — typically combined with sway-desktop-vnc for VNC-based debugging
sway-browser-vnc-moonlight:
  base: fedora
  layers:
    - sway-desktop-vnc
    - moonlight
```

## Sway Keybinding

Moonlight launches with **Alt+M** (`Mod1+m`) via sway config drop-in. Uses `--appimage-extract-and-run` for rootless container compatibility.

## Related Layers

- `/ov-layers:sunshine` -- Sunshine server (what Moonlight connects to)
- `/ov-layers:sway-desktop-vnc` -- VNC desktop for visual debugging
- `/ov:moon` -- CLI GameStream protocol commands
