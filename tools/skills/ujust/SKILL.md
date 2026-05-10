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
| Install files | `task:` |

## Usage

```yaml
# image.yml
my-image:
  layers:
    - ujust
```

## Used In Images

- `/ov-distros:bazzite-ai` (disabled)

## Related Layers
- `/ov-distros:bootc-config` — bootc system bring-up
- `/ov-distros:os-config` — bootc OS configuration sibling
- `/ov-distros:os-system-files` — system file overlay sibling

## Related Commands
- `/ov-core:shell` — run ujust inside the container
- `/ov-vm:vm` — bootc VM lifecycle for ujust-managed images

## When to Use This Skill

Use when the user asks about:

- Just task runner installation
- ujust wrapper for Universal Blue
- Justfile-based task automation
- `/usr/share/ublue-os/justfile` integration

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
