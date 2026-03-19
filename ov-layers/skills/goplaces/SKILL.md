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
  - goplaces
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## When to Use This Skill

Use when the user asks about:
- Google Places API access
- Location search tools
- The `goplaces` layer
