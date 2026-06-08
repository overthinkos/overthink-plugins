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
  - gifgrep
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:golang` — required Go toolchain dependency
- `/charly-openclaw:openclaw-full` — metalayer that includes gifgrep
- `/charly-tools:goplaces` — sibling Go-based CLI in openclaw-full

## Related Commands
- `/charly-build:build` — compiles gifgrep via the Go builder during image build
- `/charly-core:shell` — run gifgrep interactively inside a container

## When to Use This Skill

Use when the user asks about:
- GIF search and download tools
- Go-based GIF CLI
- The `gifgrep` layer

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
