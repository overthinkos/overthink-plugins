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

RPM: `tmux`

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

## Related Commands

- `/ov:tmux` — Persistent shell sessions, send commands, capture output

## When to Use This Skill

Use when the user asks about:
- Terminal multiplexer in containers
- tmux setup
- The `tmux` layer
