---
name: wl-overlay-layer
description: |
  Wayland overlay windows via gtk4-layer-shell (for screen recordings).
  Use when working with the wl-overlay layer, gtk4-layer-shell, or overlay dependencies.
---

# wl-overlay -- Wayland overlay windows via gtk4-layer-shell

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `dbus` |
| Install files | `task:`, `ov-overlay` |

## Packages

- `gtk4` (RPM)
- `gtk4-layer-shell` (RPM) — includes `Gtk4LayerShell-1.0.typelib` for GObject Introspection
- `python3-gobject` (RPM)

## Technical Details

### Python Script: `ov-overlay`

Installed to `~/.local/bin/ov-overlay` via `task:`. Dual-mode script (daemon + client) using GTK4 + gtk4-layer-shell for layer-shell protocol overlays.

**Key implementation details:**
- **Shebang**: `#!/usr/bin/python3` (system Python 3.14, NOT pixi's Python — PyGObject is an RPM package)
- **Library preload**: `CDLL("libgtk4-layer-shell.so.0")` must load before GTK4 import (registers layer-shell protocol before Wayland display init). Uses versioned soname `.so.0` (unversioned `.so` is in `-devel` package only)
- **RGBA surfaces**: `window { background: none; }` CSS applied globally at daemon startup. Without this, GTK4 on Wayland creates opaque surfaces that don't support alpha transparency
- **Cairo backgrounds**: all overlay backgrounds drawn via `Gtk.DrawingArea` + Cairo `paint()`. GTK4 CSS `background-color` with `rgba()` does not composite correctly on layer-shell windows
- **Thread safety**: socket listener runs in a separate thread; all GTK operations dispatched via `GLib.idle_add()`
- **IPC**: Unix domain socket at `/tmp/ov-overlay.sock` with JSON-line protocol

### Layer-Shell Configuration

- **Layer**: `OVERLAY` (above TOP where waybar/swaync live)
- **Keyboard mode**: `NONE` (overlays never capture input)
- **Exclusive zone**: `-1` (don't push other windows)
- **Namespace**: `ov-overlay-<name>` per overlay window

## Usage

```yaml
# image.yml — typically included via desktop metalayers
# Already part of sway-desktop, selkies-desktop
my-desktop:
  layers:
    - wl-overlay
```

```bash
ov eval wl overlay show my-image --type text --text "Hello" --name intro
ov eval wl overlay list my-image
ov eval wl overlay hide my-image --all
```

## Used In

- `/ov-selkies:sway-desktop` — sway desktop composition
- `/ov-selkies:selkies-desktop-layer` — selkies streaming desktop composition

## Used In Images

- `/ov-selkies:sway-browser-vnc` (via `sway-desktop` metalayer)
- `/ov-selkies:selkies-labwc` (via `selkies-desktop` metalayer)
- `/ov-selkies:selkies-labwc-nvidia` (via `selkies-desktop` metalayer)

## Related Layers

- `/ov-infrastructure:dbus-layer` — D-Bus session bus (dependency, required by GTK4)
- `/ov-selkies:wl-tools` — Wayland automation tools (sibling in desktop stack)
- `/ov-selkies:a11y-tools` — also uses `python3-gobject` (RPM deduplication handles overlap)
- `/ov-infrastructure:tmux-layer` — tmux (required for overlay daemon hosting)

## Related Commands

- `/ov-eval:wl-overlay` — Overlay command usage, types, recording workflow
- `/ov-eval:wl` — Compositor-agnostic desktop automation
- `/ov-eval:record` — Recording commands (overlays compose with recording)

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
