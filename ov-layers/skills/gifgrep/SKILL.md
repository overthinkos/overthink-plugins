---
name: gifgrep
description: |
  GIF search and download CLI.
  Use when working with the gifgrep layer.
---

# gifgrep -- GIF search and download

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
  - gifgrep
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:golang` — required Go toolchain dependency
- `/ov-layers:openclaw-full` — metalayer that includes gifgrep
- `/ov-layers:goplaces` — sibling Go-based CLI in openclaw-full

## Related Commands
- `/ov:build` — compiles gifgrep via the Go builder during image build
- `/ov:shell` — run gifgrep interactively inside a container

## When to Use This Skill

Use when the user asks about:
- GIF search and download tools
- Go-based GIF CLI
- The `gifgrep` layer

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov image test`, `ov test`)
