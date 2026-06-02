---
name: tmux-layer
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

## Related Skills

- `/ov-automation:tmux` — Persistent shell sessions, send commands, capture output
- `/ov-hermes:hermes-full-layer`, `/ov-selkies:sway-desktop`, `/ov-selkies:selkies-desktop-layer` — metalayers that include this
- `/ov-hermes:hermes`, `/ov-selkies:selkies-labwc`, `/ov-selkies:sway-browser-vnc` — primary consumers
- `/ov-core:shell` — interactive container shell (often wrapped in tmux)
- `/ov-image:layer` — layer authoring
- `/ov-eval:eval` — declarative testing framework (this layer tests `/usr/bin/tmux -V`)

## When to Use This Skill

Use when the user asks about:
- Terminal multiplexer in containers
- tmux setup
- The `tmux` layer
