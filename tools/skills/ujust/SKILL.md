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
# box.yml
my-image:
  layers:
    - ujust
```

## Used In Images

- `/charly-distros:bazzite`

## Related Layers
- `/charly-distros:bootc-config` — bootc system bring-up
- `/charly-distros:os-config` — bootc OS configuration sibling
- `/charly-distros:os-system-files` — system file overlay sibling

## Related Commands
- `/charly-core:shell` — run ujust inside the container
- `/charly-vm:vm` — bootc VM lifecycle for ujust-managed images

## When to Use This Skill

Use when the user asks about:

- Just task runner installation
- ujust wrapper for Universal Blue
- Justfile-based task automation
- `/usr/share/ublue-os/justfile` integration

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
