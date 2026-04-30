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

Used by `ov eval wl screenshot` — auto-detected when `grim` is available in the container.

```bash
ov eval wl screenshot <image> [output.png]
```

## Included In

- `sway-desktop` metalayer

## Used In Images

- `/ov-selkies:sway-browser-vnc` (via `sway-desktop` metalayer)
- `/ov-openclaw:openclaw-sway-browser` (via `sway-desktop` metalayer)
- `/ov-openclaw:openclaw-ollama-sway-browser` (via `sway-desktop` metalayer)

## Cross-References

- `/ov-advanced:wl` — `ov eval wl screenshot` auto-detects grim
- `/ov-selkies:wl-screenshot-pixelflux` — Alternative for selkies-desktop
- `/ov-selkies:wl-tools` — Companion layer (input, window mgmt, clipboard)

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
