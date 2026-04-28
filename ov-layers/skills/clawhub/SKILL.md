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
| Install files | `layer.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# image.yml or layer.yml
layers:
  - clawhub
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:nodejs` — Required Node.js runtime parent dependency
- `/ov-layers:openclaw-full` — Metalayer bundling clawhub with codex/gemini/claude-code
- `/ov-layers:openclaw` — OpenClaw gateway service that consumes installed skills

## Related Commands
- `/ov:build` — Builds the layer (npm global install via package.json)
- `/ov:openclaw` — OpenClaw gateway configuration where clawhub installs skills

## When to Use This Skill

Use when the user asks about:
- ClawHub skill registry
- Installing or searching OpenClaw skills
- The `clawhub` layer

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
