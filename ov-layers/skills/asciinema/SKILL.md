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

## Cross-distro coverage

RPM: `asciinema` · PAC: `asciinema` · DEB: `asciinema` — full parity.

## Usage

```yaml
# image.yml or layer.yml
layers:
  - asciinema
```

Used by `ov eval record start --mode terminal` for terminal recording sessions. Also available standalone via `asciinema rec`.

## Integration with `ov eval record`

```bash
# Start recording a terminal session
ov eval record start <image> -n demo --mode terminal

# Send commands to the recorded terminal
ov eval record cmd <image> "echo hello" -n demo

# Stop and copy to host
ov eval record stop <image> -n demo -o demo.cast

# Play back
asciinema play demo.cast
```

## Note

Also available via the `dev-tools` layer (which includes asciinema among many other tools). This standalone layer is for images that need terminal recording without the full dev-tools bundle.

## Used In Images

- `/ov-images:sway-browser-vnc` (via `sway-desktop` metalayer)
- `/ov-images:openclaw-sway-browser` (via `sway-desktop` metalayer)
- `/ov-images:openclaw-ollama-sway-browser` (via `sway-desktop` metalayer)
- `/ov-images:selkies-desktop` (via `selkies-desktop` metalayer)
- `/ov-images:selkies-desktop-nvidia` (via `selkies-desktop` metalayer)

## Related Commands

- `/ov:record` — Terminal recording via asciinema (start, stop, cmd)

## Cross-References

- `/ov:record` — `ov eval record start --mode terminal` uses asciinema
- `/ov-layers:dev-tools` — Also includes asciinema (larger layer)

## When to Use This Skill

Use when the user asks about:
- Terminal session recording
- asciinema in containers
- The `asciinema` layer

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
