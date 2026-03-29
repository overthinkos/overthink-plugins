---
name: asciinema
description: |
  Terminal session recorder (asciinema).
  Use when working with the asciinema layer.
---

# asciinema -- Terminal session recorder

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |
| Depends | none |

## Packages

RPM: `asciinema`

## What It Does

Records terminal sessions as `.cast` files (asciicast v2 format). Recordings capture timing, input, and output — can be replayed with `asciinema play`, uploaded to asciinema.org, or converted to GIF/video.

## Usage

```yaml
# images.yml or layer.yml
layers:
  - asciinema
```

Used by `ov record start --mode terminal` for terminal recording sessions. Also available standalone via `asciinema rec`.

## Integration with `ov record`

```bash
# Start recording a terminal session
ov record start <image> -n demo --mode terminal

# Send commands to the recorded terminal
ov record cmd <image> "echo hello" -n demo

# Stop and copy to host
ov record stop <image> -n demo -o demo.cast

# Play back
asciinema play demo.cast
```

## Note

Also available via the `dev-tools` layer (which includes asciinema among many other tools). This standalone layer is for images that need terminal recording without the full dev-tools bundle.

## Cross-References

- `/ov:record` — `ov record start --mode terminal` uses asciinema
- `/ov-layers:dev-tools` — Also includes asciinema (larger layer)

## When to Use This Skill

Use when the user asks about:
- Terminal session recording
- asciinema in containers
- The `asciinema` layer
