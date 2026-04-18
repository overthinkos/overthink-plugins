---
name: blogwatcher
description: |
  Blog/RSS feed monitor CLI.
  Use when working with the blogwatcher layer.
---

# blogwatcher -- Blog/RSS feed monitor

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `tasks:` |
| Depends | `golang` |

## Environment

| Variable | Value |
|----------|-------|
| `GOPATH` | `~/go` |
| PATH append | `~/go/bin` |

## Usage

```yaml
# images.yml or layer.yml
layers:
  - blogwatcher
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:golang` — Required Go runtime parent dependency
- `/ov-layers:openclaw-full` — Metalayer that bundles blogwatcher with other AI/agent CLIs

## Related Commands
- `/ov:build` — Builds the layer (Go install via a cmd task)
- `/ov:shell` — Interactive shell to run blogwatcher inside the container

## When to Use This Skill

Use when the user asks about:
- Blog or RSS feed monitoring
- Go-based CLI tools for feed watching
- The `blogwatcher` layer
