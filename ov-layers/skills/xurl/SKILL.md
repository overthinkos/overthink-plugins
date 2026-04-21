---
name: xurl
description: |
  X (Twitter) API CLI for posts, search, DMs, and media.
  Use when working with the xurl layer.
---

# xurl -- X/Twitter API CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# image.yml or layer.yml
layers:
  - xurl
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:nodejs` — runtime dependency
- `/ov-layers:openclaw-full` — metalayer that bundles xurl
- `/ov-layers:hermes` — companion social/messaging agent

## Related Commands
- `/ov:shell` — run xurl inside the container
- `/ov:secrets` — store X/Twitter API credentials

## When to Use This Skill

Use when the user asks about:
- X/Twitter API access in containers
- Posting, searching, or managing Twitter/X content
- The `xurl` layer

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov image test`, `ov test`)
