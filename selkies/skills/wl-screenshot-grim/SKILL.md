# wl-screenshot-grim - Screenshot via grim (wlr-screencopy)

## Overview

Provides `grim` for Wayland screenshot capture using the `wlr-screencopy` protocol. Works on sway and standalone wlroots compositors. **Does NOT work on selkies-desktop** (labwc nested in pixelflux can't deliver screencopy frames).

For selkies-desktop, use `wl-screenshot-pixelflux` instead.

## Candy Definition

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

Used by `charly check wl screenshot` — auto-detected when `grim` is available in the container.

```bash
charly check wl screenshot <image> [output.png]
```

## Included In

- `sway-desktop` metalayer

## Used In Boxes

- `/charly-selkies:sway-browser-vnc` (via `sway-desktop` metalayer)

## Cross-References

- `/charly-check:wl` — `charly check wl screenshot` auto-detects grim
- `/charly-selkies:wl-screenshot-pixelflux` — Alternative for selkies-desktop
- `/charly-selkies:wl-tools` — Companion candy (input, window mgmt, clipboard)

## Related

- `/charly-image:layer` — candy authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-check:check` — declarative testing (`check:` block, `charly check box`, `charly check live`)
