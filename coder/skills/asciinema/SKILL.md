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
| Install files | `charly.yml` (packages only) |
| Depends | none |

## Packages

RPM: `asciinema`

## What It Does

Records terminal sessions as `.cast` files (asciicast v2 format). Recordings capture timing, input, and output — can be replayed with `asciinema play`, uploaded to asciinema.org, or converted to GIF/video.

## Cross-distro coverage

RPM: `asciinema` · PAC: `asciinema` · DEB: `asciinema` — full parity.

## Usage

```yaml
# box or candy charly.yml
layers:
  - asciinema
```

Used by `charly eval record start --mode terminal` for terminal recording sessions. Also available standalone via `asciinema rec`.

## Integration with `charly eval record`

```bash
# Start recording a terminal session
charly eval record start <image> -n demo --mode terminal

# Send commands to the recorded terminal
charly eval record cmd <image> "echo hello" -n demo

# Stop and copy to host
charly eval record stop <image> -n demo -o demo.cast

# Play back
asciinema play demo.cast
```

## Note

Also available via the `dev-tools` layer (which includes asciinema among many other tools). This standalone layer is for images that need terminal recording without the full dev-tools bundle.

## Used In Images

- `/charly-selkies:sway-browser-vnc` (via `sway-desktop` metalayer)
- `/charly-selkies:selkies-labwc` (via `selkies-desktop` metalayer)
- `/charly-selkies:selkies-labwc-nvidia` (via `selkies-desktop` metalayer)

## Related Commands

- `/charly-eval:record` — Terminal recording via asciinema (start, stop, cmd)

## Cross-References

- `/charly-eval:record` — `charly eval record start --mode terminal` uses asciinema
- `/charly-coder:dev-tools` — Also includes asciinema (larger layer)

## When to Use This Skill

Use when the user asks about:
- Terminal session recording
- asciinema in containers
- The `asciinema` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
