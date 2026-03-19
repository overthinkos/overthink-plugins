---
name: playwright
description: |
  Playwright browser automation (OpenClaw AI snapshots).
  Use when working with the playwright layer.
---

# playwright -- Browser automation

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# images.yml or layer.yml
layers:
  - playwright
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## When to Use This Skill

Use when the user asks about:
- Playwright browser automation in containers
- OpenClaw AI snapshot support
- The `playwright` layer
