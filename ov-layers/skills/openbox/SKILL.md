---
name: openbox
description: |
  Openbox lightweight X11 window manager with keybindings and desktop support.
  Use when working with Openbox WM in X11 desktop containers.
---

# openbox -- Lightweight X11 window manager

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `xorg-headless` |
| Service | `openbox` (supervisord, priority 12) |
| Packages | openbox, xterm |
| Install files | `tasks:`, `openbox-wrapper`, `rc.xml` |

## Keybindings

| Key | Action |
|-----|--------|
| Alt+Return | Open xfce4-terminal |
| Alt+D | Open xterm |
| Alt+Q | Close window |
| Alt+F | Toggle fullscreen |
| Alt+Space | Toggle maximize |
| Alt+Tab | Next window |
| Alt+1-5 | Switch desktop |
| Alt+Shift+E | Exit |

## Related Layers

- `/ov-layers:xorg-headless` -- X11 display server (dependency)
- `/ov-layers:chrome-x11` -- Chrome on X11 (launched via openbox autostart)
- `/ov-layers:x11-desktop` -- desktop composition

## Used In Images

Not used in any current image definition. Part of the `x11-desktop` metalayer composition.

## When to Use This Skill

Use when working with Openbox WM configuration or X11 desktop containers.
