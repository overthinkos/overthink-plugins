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
| Install files | `tasks:`, `config.json`, `style.css`, `waybar-wrapper`, `sway-autotile` |

## Packages

- `waybar` (RPM)

Fonts (JetBrains Mono, Symbols Nerd Font) provided by the `desktop-fonts` layer in metalayers.

## Configuration

Unified config shared with the `waybar-labwc` layer (Catppuccin Mocha, bottom bar):

### Modules

| Position | Module | Purpose |
|----------|--------|---------|
| Left | `custom/chrome` | (Re)start Chrome button — calls `chrome-restart` (see `/ov-selkies:chrome`) |
| Center | `wlr/taskbar` | Running app icons — click to activate, middle-click to close, right-click to minimize |
| Right | `custom/notification` | swaync notification toggle — click to open notification panel (see `/ov-selkies:swaync`) |

### Style

Catppuccin Mocha theme — semi-transparent dark background, JetBrains Mono + Symbols Nerd Font.

## Key Files

- `waybar-wrapper` — Waits for Wayland socket, discovers SWAYSOCK, then exec waybar
- `sway-autotile` — Subscribes to sway window events, auto-tiles windows restored from scratchpad
- `config.json` — Unified module layout (same as waybar-labwc layer)
- `style.css` — Catppuccin Mocha styling (same as waybar-labwc layer)

## Used In Images

Part of `/ov-selkies:sway-desktop` composition.

## Related Layers

- `/ov-selkies:sway` -- compositor dependency
- `/ov-selkies:sway-desktop` -- composition that includes waybar
- `/ov-selkies:swaync` -- notification daemon (notification bell module)
- `/ov-selkies:pavucontrol` -- volume control (pulseaudio on-click)
- `/ov-selkies:desktop-fonts` -- JetBrains Mono + Nerd Font icons

## Related Commands
- `/ov-advanced:wl` — interact with the Wayland session hosting waybar
- `/ov-core:shell` — restart waybar via supervisorctl

## When to Use This Skill

Use when the user asks about:

- Waybar configuration or styling
- Status bar in Sway desktop
- Automatic window tiling
- Desktop UI customization

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
