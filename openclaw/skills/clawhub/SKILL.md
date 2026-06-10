---
name: clawhub
description: |
  ClawHub CLI for searching and installing OpenClaw skills.
  Use when working with the clawhub layer.
---

# clawhub -- ClawHub skill registry CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `charly.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# box or candy charly.yml
layers:
  - clawhub
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:nodejs` — Required Node.js runtime parent dependency
- `/charly-openclaw:openclaw-full` — Metalayer bundling clawhub with codex/gemini/claude-code
- `/charly-openclaw:openclaw` — OpenClaw gateway service that consumes installed skills

## Related Commands
- `/charly-build:build` — Builds the layer (npm global install via package.json)
- `/charly-automation:openclaw-deploy` — OpenClaw gateway configuration where clawhub installs skills

## When to Use This Skill

Use when the user asks about:
- ClawHub skill registry
- Installing or searching OpenClaw skills
- The `clawhub` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
