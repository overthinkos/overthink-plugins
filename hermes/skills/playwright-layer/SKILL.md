---
name: playwright-layer
description: |
  Playwright browser automation (OpenClaw AI snapshots).
  Use when working with the playwright layer.
---

# playwright -- Browser automation

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `candy.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# box.yml or candy.yml
layers:
  - playwright
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:nodejs` -- runtime dependency
- `/charly-openclaw:openclaw-full` -- parent metalayer that bundles playwright
- `/charly-selkies:chrome` -- browser commonly driven by playwright

## Related Commands
- `/charly-eval:cdp` -- Chrome DevTools Protocol automation alternative
- `/charly-automation:openclaw-deploy` -- snapshot/skill configuration

## When to Use This Skill

Use when the user asks about:
- Playwright browser automation in containers
- OpenClaw AI snapshot support
- The `playwright` layer

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
