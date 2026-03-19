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
| Install files | `layer.yml`, `user.yml` |
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
  - gogcli
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## When to Use This Skill

Use when the user asks about:
- Google Workspace CLI access (Gmail, Calendar, Drive, Contacts, Sheets, Docs)
- Google API tools in containers
- The `gogcli` layer
