---
name: tmux
description: |
  Terminal multiplexer.
  Use when working with the tmux layer.
---

# tmux -- Terminal multiplexer

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |
| Depends | none |

## Packages

RPM: `tmux` · PAC: `tmux` · DEB: `tmux` — full cross-distro parity.

## Usage

```yaml
# image.yml or layer.yml
layers:
  - tmux
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Skills

- `/ov:tmux` — Persistent shell sessions, send commands, capture output
- `/ov-layers:hermes-full`, `/ov-layers:sway-desktop`, `/ov-layers:selkies-desktop` — metalayers that include this
- `/ov-images:hermes`, `/ov-images:selkies-desktop`, `/ov-images:sway-browser-vnc` — primary consumers
- `/ov:shell` — interactive container shell (often wrapped in tmux)
- `/ov:layer` — layer authoring
- `/ov:test` — declarative testing framework (this layer tests `/usr/bin/tmux -V`)

## When to Use This Skill

Use when the user asks about:
- Terminal multiplexer in containers
- tmux setup
- The `tmux` layer
