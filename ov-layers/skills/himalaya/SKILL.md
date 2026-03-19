---
name: himalaya
description: |
  Himalaya email CLI (IMAP/SMTP).
  Use when working with the himalaya layer.
---

# himalaya -- Email CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `user.yml` |
| Depends | `rust` |

## Usage

```yaml
# images.yml or layer.yml
layers:
  - himalaya
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## When to Use This Skill

Use when the user asks about:
- Email CLI access (IMAP/SMTP)
- Himalaya email client in containers
- The `himalaya` layer
