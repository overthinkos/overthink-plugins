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
# images.yml or layer.yml
layers:
  - clawhub
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## When to Use This Skill

Use when the user asks about:
- ClawHub skill registry
- Installing or searching OpenClaw skills
- The `clawhub` layer
