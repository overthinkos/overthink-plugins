---
name: os-system-files
description: |
  System files overlay and justfile imports for bootc images. Copies system_files to root filesystem.
  Use when working with system file overlays, justfile imports, or bazzite-ai configuration.
---

# os-system-files -- system files overlay

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `tasks:` |

## Usage

```yaml
# image.yml
my-bootc-image:
  bootc: true
  layers:
    - os-system-files
```

## Used In Images

- `/ov-images:bazzite-ai` (disabled)

## Related Layers
- `/ov-layers:os-config` -- companion bootc system configuration layer
- `/ov-layers:bootc-base` -- bootc image base composition

## Related Commands
- `/ov:build` -- build the bootc image that consumes this layer
- `/ov:vm` -- boot the resulting bootc disk image

## When to Use This Skill

Use when the user asks about:

- System file overlays in bootc images
- Copying system_files directory to root filesystem
- Justfile imports for bazzite-ai
- `/usr/share/bazzite-ai/just/` configuration
- `60-custom.just` generation

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
