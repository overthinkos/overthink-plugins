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
| Install files | `charly.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# box or candy charly.yml
candy:
  - xurl
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:nodejs` — runtime dependency
- `/charly-openclaw:openclaw-full` — metalayer that bundles xurl
- `/charly-hermes:hermes` — companion social/messaging agent

## Related Commands
- `/charly-core:shell` — run xurl inside the container
- `/charly-build:secrets` — store X/Twitter API credentials

## When to Use This Skill

Use when the user asks about:
- X/Twitter API access in containers
- Posting, searching, or managing Twitter/X content
- The `xurl` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
