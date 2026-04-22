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
| Install files | `layer.yml` (packages only) |
| Depends | none |

## Packages

RPM: `wf-recorder`

## What It Does

Native Wayland screen recorder for wlroots-based compositors (sway, labwc, etc.). Captures frames via the `wlr-screencopy` protocol and encodes to MP4/MKV using system ffmpeg. Supports audio capture with `--audio`.

**Note:** Does NOT work on selkies-desktop — use `wl-record-pixelflux` instead (labwc nested in pixelflux can't deliver screencopy frames).

## Usage

```yaml
# image.yml or layer.yml
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

## Integration with `ov test record`

```bash
# Start recording sway desktop (auto-detects wf-recorder)
ov test record start sway-browser-vnc -n demo --mode desktop

# Interact with desktop
ov test wl click sway-browser-vnc 640 360
ov test record cmd sway-browser-vnc "neofetch" -n demo

# Stop and copy to host
ov test record stop sway-browser-vnc -n demo -o demo.mp4
```

## Included In

- `sway-desktop` metalayer (and all images using sway-desktop-vnc)

## Used In Images

- `/ov-images:sway-browser-vnc` (via `sway-desktop` metalayer)
- `/ov-images:openclaw-sway-browser` (via `sway-desktop` metalayer)
- `/ov-images:openclaw-ollama-sway-browser` (via `sway-desktop` metalayer)

## Cross-References

- `/ov:record` — `ov test record start --mode desktop` auto-detects wf-recorder
- `/ov-layers:wl-record-pixelflux` — Alternative for selkies-desktop (pixelflux pipeline)
- `/ov-layers:wl-screenshot-grim` — Screenshot companion (same wlr-screencopy protocol)
- `/ov-layers:sway-desktop` — Parent metalayer

## When to Use This Skill

Use when the user asks about:
- Desktop video recording on sway
- wf-recorder in containers
- The `wf-recorder` layer

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov image test`, `ov test`)
