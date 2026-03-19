---
name: ujust
description: |
  Just task runner with ujust wrapper for Universal Blue justfile conventions.
  Use when working with just/ujust task runners or ublue-os justfile integration.
---

# ujust -- just task runner with ujust wrapper

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `root.yml` |

## Usage

```yaml
# images.yml
my-image:
  layers:
    - ujust
```

## Used In Images

- `/ov-images:bazzite-ai` (disabled)

## When to Use This Skill

Use when the user asks about:

- Just task runner installation
- ujust wrapper for Universal Blue
- Justfile-based task automation
- `/usr/share/ublue-os/justfile` integration
