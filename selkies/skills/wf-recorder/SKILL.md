---
name: wf-recorder
description: |
  Wayland screen recorder for wlroots compositors.
  Use when working with the wf-recorder candy.
---

# wf-recorder -- Wayland screen recorder

## Candy Properties

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
# a box composing this candy — the candy list is a child node
my-box:
  candy:
    base: fedora
  my-box-candy:
    candy:
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

## Integration with `charly check record`

```bash
# Start recording sway desktop (auto-detects wf-recorder)
charly check record start sway-browser-vnc -n demo --mode desktop

# Interact with desktop
charly check wl click sway-browser-vnc 640 360
charly check record cmd sway-browser-vnc "neofetch" -n demo

# Stop and copy to host
charly check record stop sway-browser-vnc -n demo -o demo.mp4
```

## Included In

- `sway-desktop` metalayer (and all boxes using sway-desktop-vnc)

## Used In Boxes

- `/charly-selkies:sway-browser-vnc` (via `sway-desktop` metalayer)

## Cross-References

- `/charly-check:record` — `charly check record start --mode desktop` auto-detects wf-recorder
- `/charly-selkies:wl-record-pixelflux` — Alternative for selkies-desktop (pixelflux pipeline)
- `/charly-selkies:wl-screenshot-grim` — Screenshot companion (same wlr-screencopy protocol)
- `/charly-selkies:sway-desktop` — Parent metalayer

## When to Use This Skill

Use when the user asks about:
- Desktop video recording on sway
- wf-recorder in containers
- The `wf-recorder` candy

## Related

- `/charly-image:layer` — candy authoring reference (`charly.yml` schema, `run:`/`check:` step verbs, service declarations)
- `/charly-check:check` — declarative testing (`check:` block, `charly check box`, `charly check live`)
