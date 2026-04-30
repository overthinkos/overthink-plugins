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
- `/ov-coder:golang` — required Go toolchain dependency
- `/ov-openclaw:openclaw-full` — metalayer that includes gifgrep
- `/ov-foundation:goplaces` — sibling Go-based CLI in openclaw-full

## Related Commands
- `/ov-build:build` — compiles gifgrep via the Go builder during image build
- `/ov-core:shell` — run gifgrep interactively inside a container

## When to Use This Skill

Use when the user asks about:
- GIF search and download tools
- Go-based GIF CLI
- The `gifgrep` layer

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
