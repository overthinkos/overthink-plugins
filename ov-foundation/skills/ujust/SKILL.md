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
# image.yml
my-image:
  layers:
    - ujust
```

## Used In Images

- `/ov-foundation:bazzite-ai` (disabled)

## Related Layers
- `/ov-foundation:bootc-config` — bootc system bring-up
- `/ov-foundation:os-config` — bootc OS configuration sibling
- `/ov-foundation:os-system-files` — system file overlay sibling

## Related Commands
- `/ov-core:shell` — run ujust inside the container
- `/ov-advanced:vm` — bootc VM lifecycle for ujust-managed images

## When to Use This Skill

Use when the user asks about:

- Just task runner installation
- ujust wrapper for Universal Blue
- Justfile-based task automation
- `/usr/share/ublue-os/justfile` integration

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
