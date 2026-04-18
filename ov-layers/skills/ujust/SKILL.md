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
| Install files | `tasks:` |

## Usage

```yaml
# images.yml
my-image:
  layers:
    - ujust
```

## Used In Images

- `/ov-images:bazzite-ai` (disabled)

## Related Layers
- `/ov-layers:bootc-config` — bootc system bring-up
- `/ov-layers:os-config` — bootc OS configuration sibling
- `/ov-layers:os-system-files` — system file overlay sibling

## Related Commands
- `/ov:shell` — run ujust inside the container
- `/ov:vm` — bootc VM lifecycle for ujust-managed images

## When to Use This Skill

Use when the user asks about:

- Just task runner installation
- ujust wrapper for Universal Blue
- Justfile-based task automation
- `/usr/share/ublue-os/justfile` integration
