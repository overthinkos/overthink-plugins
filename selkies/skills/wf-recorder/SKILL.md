---
name: wf-recorder
description: |
  Wayland screen recorder for wlroots compositors.
  Use when working with the wf-recorder layer.
---

# wf-recorder -- Wayland screen recorder

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `charly.yml` (packages only) |
| Depends | none |

## Packages

RPM: `wf-recorder`

## What It Does

Native Wayland screen recorder for wlroots-based compositors (sway, labwc, etc.). Captures frames via the `wlr-screencopy` protocol and encodes to MP4/MKV using system ffmpeg. Supports audio capture with `--audio`.

**Note:** Does NOT work on selkies-desktop — use `wl-record-pixelflux` instead (labwc nested in pixelflux can't deliver screencopy frames).

## Usage

```yaml
# box or candy charly.yml
layers:
  - wf-recorder
```

## CLI Usage

```bash
# Direct usage (inside container)
wf-recorder -f output.mp4              # Record to MP4
wf-recorder -f output.mp4 --audio      # With audio
wf-recorder -f output.mp4 -r 60        # 60fps
# Stop with Ctrl-C
```

## Integration with `charly eval record`

```bash
# Start recording sway desktop (auto-detects wf-recorder)
charly eval record start sway-browser-vnc -n demo --mode desktop

# Interact with desktop
charly eval wl click sway-browser-vnc 640 360
charly eval record cmd sway-browser-vnc "neofetch" -n demo

# Stop and copy to host
charly eval record stop sway-browser-vnc -n demo -o demo.mp4
```

## Included In

- `sway-desktop` metalayer (and all images using sway-desktop-vnc)

## Used In Images

- `/charly-selkies:sway-browser-vnc` (via `sway-desktop` metalayer)

## Cross-References

- `/charly-eval:record` — `charly eval record start --mode desktop` auto-detects wf-recorder
- `/charly-selkies:wl-record-pixelflux` — Alternative for selkies-desktop (pixelflux pipeline)
- `/charly-selkies:wl-screenshot-grim` — Screenshot companion (same wlr-screencopy protocol)
- `/charly-selkies:sway-desktop` — Parent metalayer

## When to Use This Skill

Use when the user asks about:
- Desktop video recording on sway
- wf-recorder in containers
- The `wf-recorder` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
