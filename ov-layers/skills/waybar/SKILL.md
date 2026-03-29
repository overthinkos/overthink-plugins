---
name: waybar
description: |
  Waybar status bar and sway-autotile for the Sway desktop compositor.
  Use when working with Waybar configuration, status bar, or automatic tiling.
---

# waybar -- Status bar and auto-tiling for Sway

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `sway` |
| Service | `waybar` (supervisord, priority 15), `sway-autotile` (priority 16) |
| Install files | `user.yml`, `config.json`, `style.css`, `waybar-wrapper`, `sway-autotile` |

## Packages

- `waybar` (RPM)

Fonts (JetBrains Mono, Symbols Nerd Font) provided by the `desktop-fonts` layer in metalayers.

## Configuration

Unified config shared with the `waybar-labwc` layer (Catppuccin Mocha, top bar):

### Modules

| Position | Module | Purpose |
|----------|--------|---------|
| Left | `sway/workspaces` | Workspace indicator |
| Left | `sway/mode` | Sway mode indicator |
| Left | `wlr/taskbar` | Running windows (click to activate) |
| Center | `clock` | Time with calendar tooltip (Europe/Vienna) |
| Right | `cpu`, `memory`, `disk` | System monitors |
| Right | `network` | Container IP and bandwidth |
| Right | `pulseaudio` | Volume control (click opens pavucontrol) |
| Right | `tray` | System tray |
| Right | `custom/notification` | swaync notification bell |

### Style

Catppuccin Mocha theme — semi-transparent dark background, JetBrains Mono + Symbols Nerd Font.

## Key Files

- `waybar-wrapper` — Waits for Wayland socket, discovers SWAYSOCK, then exec waybar
- `sway-autotile` — Subscribes to sway window events, auto-tiles windows restored from scratchpad
- `config.json` — Unified module layout (same as waybar-labwc layer)
- `style.css` — Catppuccin Mocha styling (same as waybar-labwc layer)

## Used In Images

Part of `/ov-layers:sway-desktop` composition.

## Related Layers

- `/ov-layers:sway` -- compositor dependency
- `/ov-layers:sway-desktop` -- composition that includes waybar
- `/ov-layers:swaync` -- notification daemon (notification bell module)
- `/ov-layers:pavucontrol` -- volume control (pulseaudio on-click)
- `/ov-layers:desktop-fonts` -- JetBrains Mono + Nerd Font icons

## When to Use This Skill

Use when the user asks about:

- Waybar configuration or styling
- Status bar in Sway desktop
- Automatic window tiling
- Desktop UI customization
