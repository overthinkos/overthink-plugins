---
name: goplaces
description: |
  Google Places API CLI for location search.
  Use when working with the goplaces layer.
---

# goplaces -- Google Places API CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `candy.yml`, `task:` |
| Depends | `golang` |

## Environment

| Variable | Value |
|----------|-------|
| `GOPATH` | `~/go` |
| PATH append | `~/go/bin` |

## Usage

```yaml
# box.yml or candy.yml
layers:
  - goplaces
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:golang` — required Go toolchain dependency
- `/charly-openclaw:openclaw-full` — metalayer that includes goplaces
- `/charly-tools:gogcli` — sibling Google API CLI in openclaw-full

## Related Commands
- `/charly-build:secrets` — provision Google Places API key for the CLI
- `/charly-core:shell` — run goplaces interactively inside a container
- `/charly-build:build` — compiles goplaces via the Go builder during image build

## When to Use This Skill

Use when the user asks about:
- Google Places API access
- Location search tools
- The `goplaces` layer

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
