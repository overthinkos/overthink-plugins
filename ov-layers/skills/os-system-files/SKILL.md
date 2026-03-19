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
| Install files | `root.yml` |

## Usage

```yaml
# images.yml
my-bootc-image:
  bootc: true
  layers:
    - os-system-files
```

## Used In Images

- `/ov-images:bazzite-ai` (disabled)

## When to Use This Skill

Use when the user asks about:

- System file overlays in bootc images
- Copying system_files directory to root filesystem
- Justfile imports for bazzite-ai
- `/usr/share/bazzite-ai/just/` configuration
- `60-custom.just` generation
