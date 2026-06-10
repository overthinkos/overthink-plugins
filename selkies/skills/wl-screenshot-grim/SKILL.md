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

Used by `charly eval wl screenshot` — auto-detected when `grim` is available in the container.

```bash
charly eval wl screenshot <image> [output.png]
```

## Included In

- `sway-desktop` metalayer

## Used In Images

- `/charly-selkies:sway-browser-vnc` (via `sway-desktop` metalayer)

## Cross-References

- `/charly-eval:wl` — `charly eval wl screenshot` auto-detects grim
- `/charly-selkies:wl-screenshot-pixelflux` — Alternative for selkies-desktop
- `/charly-selkies:wl-tools` — Companion layer (input, window mgmt, clipboard)

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
