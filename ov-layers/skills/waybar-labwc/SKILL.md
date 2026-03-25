# Layer: waybar-labwc

Waybar taskbar panel adapted for labwc compositor (not sway). Provides app launcher buttons and a window taskbar at the bottom of the screen.

## Architecture

Waybar connects to `wayland-0` (labwc's socket), NOT `wayland-1` (pixelflux). This is critical for:
- **Layer-shell exclusive zones** — Waybar reserves space at the bottom, windows don't overlap it
- **wlr-foreign-toplevel-management** — Waybar's taskbar can see running windows in labwc

## Dependencies

- `labwc`

## Packages

- `waybar` — Status bar
- `dejavu-sans-mono-fonts`, `google-noto-sans-mono-fonts` — Fonts

## Service (supervisord)

| Service | Priority | Purpose |
|---------|----------|---------|
| `waybar` | 15 | Bottom panel (after labwc at 12, before nginx at 18) |

## Configuration

### Modules

| Position | Module | Purpose |
|----------|--------|---------|
| Left | `custom/chrome` | Launch Chrome (with CDP flags) |
| Left | `custom/terminal` | Launch foot terminal |
| Left | `custom/files` | Launch thunar file manager |
| Center | `wlr/taskbar` | Running windows (click to activate, right-click to close) |
| Right | `clock` | Time (HH:MM:SS, 1s interval) |

### Style

Catppuccin Mocha theme — dark background (#11111b), high contrast buttons with hover effects.

## Key Files

- `waybar-labwc-wrapper` — Waits for `wayland-0` socket, sets `WAYLAND_DISPLAY=wayland-0` explicitly
- `config.jsonc` — Module layout and app launch commands
- `style.css` — Catppuccin Mocha styling

## Note: Difference from `waybar` Layer

The existing `waybar` layer depends on sway and uses `swaymsg` for everything. This `waybar-labwc` variant uses direct app launch commands (no sway dependency).
