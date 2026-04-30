# Layer: waybar-labwc

Waybar status bar adapted for labwc compositor (not sway). Uses the same unified config as the `waybar` layer — sway-specific modules (workspaces, mode) auto-hide on labwc since `SWAYSOCK` is not set.

## Architecture

Waybar connects to `wayland-0` (labwc's socket), NOT `wayland-1` (pixelflux). This is critical for:
- **Layer-shell exclusive zones** — Waybar reserves space at the top, windows don't overlap it
- **wlr-foreign-toplevel-management** — Waybar's taskbar can see running windows in labwc

## Dependencies

- `labwc`

## Packages

- `waybar` (RPM)

Fonts (JetBrains Mono, Symbols Nerd Font) provided by the `desktop-fonts` layer in metalayers.

## Service (supervisord)

| Service | Priority | Purpose |
|---------|----------|---------|
| `waybar` | 15 | Bottom panel (after labwc at 12, swaync at 14, before nginx at 18) |

## Configuration

Unified config shared with the `waybar` layer (Catppuccin Mocha, bottom bar):

### Modules

| Position | Module | Purpose |
|----------|--------|---------|
| Left | `custom/chrome` | (Re)start Chrome button — calls `chrome-restart` (see `/ov-selkies:chrome`) |
| Center | `wlr/taskbar` | Running app icons — click to activate, middle-click to close, right-click to minimize |
| Right | `custom/notification` | swaync notification toggle — click to open notification panel (see `/ov-selkies:swaync`) |

### Style

Catppuccin Mocha theme — semi-transparent dark background, JetBrains Mono + Symbols Nerd Font.

## Key Files

- `waybar-labwc-wrapper` — Waits for `wayland-0` socket, sets `WAYLAND_DISPLAY=wayland-0` explicitly
- `config.json` — Unified module layout (same as waybar layer)
- `style.css` — Catppuccin Mocha styling (same as waybar layer)

## Used In Images

- `/ov-selkies:selkies-desktop` (via `selkies-desktop` metalayer)
- `/ov-selkies:selkies-desktop-nvidia` (via `selkies-desktop` metalayer)

## Note: Difference from `waybar` Layer

Both layers use the same config. The only differences are:
- **Dependency:** `labwc` instead of `sway`
- **Wrapper:** Explicitly pins `WAYLAND_DISPLAY=wayland-0` (labwc socket, not pixelflux's wayland-1)
- **No sway-autotile:** Autotiling is sway-specific

## Related

- `/ov-selkies:waybar` — the sway-native sibling with the same config
- `/ov-selkies:labwc` — compositor this layer targets
- `/ov-selkies:selkies-desktop` — metalayer that composes this layer
- `/ov-selkies:swaync` + `/ov-selkies:chrome` — consumers of the status-bar modules

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
