# wl-screenshot-grim - Screenshot via grim (wlr-screencopy)

## Overview

Provides `grim` for Wayland screenshot capture using the `wlr-screencopy` protocol. Works on sway and standalone wlroots compositors. **Does NOT work on selkies-desktop** (labwc nested in pixelflux can't deliver screencopy frames).

For selkies-desktop, use `wl-screenshot-pixelflux` instead.

## Layer Definition

```yaml
rpm:
  packages:
    - grim
```

## Key Properties

| Property | Value |
|----------|-------|
| Depends | None |
| Packages | `grim` |
| Protocol | `wlr-screencopy-unstable-v1`, `ext-image-copy-capture-v1` |

## Usage

Used by `ov wl screenshot` — auto-detected when `grim` is available in the container.

```bash
ov wl screenshot <image> [output.png]
```

## Included In

- `sway-desktop` metalayer

## Cross-references

- `/ov:wl` — `ov wl screenshot` auto-detects grim
- `/ov-layers:wl-screenshot-pixelflux` — Alternative for selkies-desktop
- `/ov-layers:wl-tools` — Companion layer (input, window mgmt, clipboard)
