# a11y-tools - AT-SPI2 Accessibility Introspection

## Overview

Provides Python AT-SPI2 bindings for querying the accessibility tree of GTK, Qt, and Chrome applications. Enables element-based automation — find buttons, menus, and text fields by name/role instead of pixel coordinates.

Used by `ov wl atspi tree/find/click`.

## Layer Definition

```yaml
depends:
  - dbus

rpm:
  packages:
    - python3-pyatspi
    - python3-gobject
```

## Key Properties

| Property | Value |
|----------|-------|
| Depends | `dbus` (AT-SPI2 requires D-Bus session bus) |
| Packages | `python3-pyatspi`, `python3-gobject` |
| Python | Uses `/usr/bin/python3` (system Python 3.14, NOT pixi's Python 3.13) |

## Important: System Python Required

The RPM packages install to system Python (`/usr/bin/python3`). Containers with pixi environments have pixi's python3 first in PATH, but pixi's python3 doesn't see system RPM packages. The `ov wl atspi` command explicitly uses `/usr/bin/python3` to ensure AT-SPI2 bindings are found.

## What You Can Query

- **tree** — Full accessibility tree (app name, window title, widget name/role/position/actions)
- **find** — Search by name, role, or `name:role` pattern
- **click** — Find an element and invoke its click/press/activate action

## Chrome Accessibility

Chrome needs `--force-renderer-accessibility` flag to expose DOM elements via AT-SPI2. The `labwc/autostart` in selkies-desktop includes this flag.

## Included In

- `selkies-desktop` metalayer

## Used In Images

- `/ov-images:selkies-desktop` (via `selkies-desktop` metalayer)
- `/ov-images:selkies-desktop-nvidia` (via `selkies-desktop` metalayer)
- `/ov-images:selkies-desktop-hermes` (via `selkies-desktop` metalayer)
- `/ov-images:selkies-desktop-hermes-jupyter` (via `selkies-desktop` metalayer)

## Cross-References

- `/ov:wl` — `ov wl atspi tree/find/click` commands
- `/ov-layers:dbus` — Required dependency (D-Bus session bus)
- `/ov-layers:selkies-desktop` — Desktop metalayer that includes this layer
