---
name: gogcli
description: |
  Google Workspace CLI (Gmail, Calendar, Drive, Contacts, Sheets, Docs).
  Use when working with the gogcli layer.
---

# gogcli -- Google Workspace CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `task:` |
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
  - gogcli
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/ov-coder:golang` — required Go toolchain dependency
- `/ov-openclaw:openclaw-full` — metalayer that includes gogcli
- `/ov-tools:goplaces` — sibling Google API CLI in openclaw-full

## Related Commands
- `/ov-build:secrets` — provision Google OAuth credentials for gogcli
- `/ov-core:shell` — run gogcli interactively inside a container
- `/ov-build:build` — compiles gogcli via the Go builder during image build

## When to Use This Skill

Use when the user asks about:
- Google Workspace CLI access (Gmail, Calendar, Drive, Contacts, Sheets, Docs)
- Google API tools in containers
- The `gogcli` layer

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
