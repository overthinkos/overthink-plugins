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
| Install files | `layer.yml`, `tasks:` |
| Depends | `golang` |

## Environment

| Variable | Value |
|----------|-------|
| `GOPATH` | `~/go` |
| PATH append | `~/go/bin` |

## Usage

```yaml
# image.yml or layer.yml
layers:
  - goplaces
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:golang` — required Go toolchain dependency
- `/ov-layers:openclaw-full` — metalayer that includes goplaces
- `/ov-layers:gogcli` — sibling Google API CLI in openclaw-full

## Related Commands
- `/ov:secrets` — provision Google Places API key for the CLI
- `/ov:shell` — run goplaces interactively inside a container
- `/ov:build` — compiles goplaces via the Go builder during image build

## When to Use This Skill

Use when the user asks about:
- Google Places API access
- Location search tools
- The `goplaces` layer

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov image test`, `ov test`)
