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
| Install files | `charly.yml`, `task:` |
| Depends | `golang` |

## Environment

| Variable | Value |
|----------|-------|
| `GOPATH` | `~/go` |
| PATH append | `~/go/bin` |

## Usage

```yaml
# box or candy charly.yml
candy:
  - blogwatcher
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:golang` — Required Go runtime parent dependency
- `/charly-openclaw:openclaw-full` — Metalayer that bundles blogwatcher with other AI/agent CLIs

## Related Commands
- `/charly-build:build` — Builds the layer (Go install via a cmd task)
- `/charly-core:shell` — Interactive shell to run blogwatcher inside the container

## When to Use This Skill

Use when the user asks about:
- Blog or RSS feed monitoring
- Go-based CLI tools for feed watching
- The `blogwatcher` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
