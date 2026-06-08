---
name: os-system-files
description: |
  System files overlay and justfile imports for bootc images. Copies system_files to root filesystem.
  Use when working with system file overlays, justfile imports, or bazzite configuration.
---

# os-system-files -- system files overlay

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `task:` |

## Usage

```yaml
# box.yml
my-bootc-image:
  bootc: true
  layers:
    - os-system-files
```

## Used In Images

- `/charly-distros:bazzite`

## Related Layers
- `/charly-distros:os-config` -- companion bootc system configuration layer
- `/charly-distros:bootc-base` -- bootc image base composition

## Related Commands
- `/charly-build:build` -- build the bootc image that consumes this layer
- `/charly-vm:vm` -- boot the resulting bootc disk image

## When to Use This Skill

Use when the user asks about:

- System file overlays in bootc images
- Copying system_files directory to root filesystem
- Justfile imports for bazzite
- `/usr/share/bazzite/just/` configuration
- `60-custom.just` generation

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
