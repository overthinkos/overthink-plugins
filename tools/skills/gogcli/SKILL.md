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
layers:
  - gogcli
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:golang` — required Go toolchain dependency
- `/charly-openclaw:openclaw-full` — metalayer that includes gogcli
- `/charly-tools:goplaces` — sibling Google API CLI in openclaw-full

## Related Commands
- `/charly-build:secrets` — provision Google OAuth credentials for gogcli
- `/charly-core:shell` — run gogcli interactively inside a container
- `/charly-build:build` — compiles gogcli via the Go builder during image build

## When to Use This Skill

Use when the user asks about:
- Google Workspace CLI access (Gmail, Calendar, Drive, Contacts, Sheets, Docs)
- Google API tools in containers
- The `gogcli` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
