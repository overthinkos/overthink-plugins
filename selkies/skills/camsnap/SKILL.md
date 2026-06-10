---
name: camsnap
description: |
  RTSP/ONVIF camera snapshot and clip CLI.
  Use when working with the camsnap candy.
---

# camsnap -- Camera snapshot CLI

## Candy Properties

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
  - camsnap
```

## Used In Boxes

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Candies
- `/charly-coder:golang` — Required Go runtime parent dependency
- `/charly-openclaw:openclaw-full` — Metalayer that bundles camsnap with other agent CLIs

## Related Commands
- `/charly-build:build` — Builds the candy (Go install via a cmd task)
- `/charly-core:shell` — Interactive shell to run camsnap inside the container

## When to Use This Skill

Use when the user asks about:
- RTSP or ONVIF camera snapshots
- Camera snapshot/clip tools
- The `camsnap` candy

## Related

- `/charly-image:layer` — candy authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
