---
name: playwright-layer
description: |
  Playwright browser automation (OpenClaw AI snapshots).
  Use when working with the playwright candy.
---

# playwright -- Browser automation

## Candy Properties

| Property | Value |
|----------|-------|
| Install files | `charly.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# box or candy charly.yml
candy:
  - playwright
```

## Used In Boxes

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Candies
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
- The `playwright` candy

## Related

- `/charly-image:layer` — candy authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
