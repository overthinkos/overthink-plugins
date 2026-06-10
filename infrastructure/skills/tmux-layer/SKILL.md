---
name: tmux-layer
description: |
  Terminal multiplexer.
  Use when working with the tmux candy.
---

# tmux -- Terminal multiplexer

## Candy Properties

| Property | Value |
|----------|-------|
| Install files | `charly.yml` (packages only) |
| Depends | none |

## Packages

RPM: `tmux` · PAC: `tmux` · DEB: `tmux` — full cross-distro parity.

## Usage

```yaml
# box or candy charly.yml
candy:
  - tmux
```

## Used In Boxes

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Skills

- `/charly-automation:tmux` — Persistent shell sessions, send commands, capture output
- `/charly-hermes:hermes-full-layer`, `/charly-selkies:sway-desktop`, `/charly-selkies:selkies-desktop-layer` — metalayers that include this
- `/charly-hermes:hermes`, `/charly-selkies:selkies-labwc`, `/charly-selkies:sway-browser-vnc` — primary consumers
- `/charly-core:shell` — interactive container shell (often wrapped in tmux)
- `/charly-image:layer` — candy authoring
- `/charly-eval:eval` — declarative testing framework (this candy tests `/usr/bin/tmux -V`)

## When to Use This Skill

Use when the user asks about:
- Terminal multiplexer in containers
- tmux setup
- The `tmux` candy
