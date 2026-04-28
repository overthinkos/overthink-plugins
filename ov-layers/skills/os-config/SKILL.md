---
name: os-config
description: |
  OS system configuration for bootc images: SDDM/KDE cleanup, systemd preset, /opt permissions, initramfs rebuild.
  Use when working with bootc OS configuration, systemd presets, or initramfs in bootable containers.
---

# os-config -- OS system configuration

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
    - os-config
```

## Used In Images

- `/ov-images:bazzite-ai` (disabled)

## Related Layers
- `/ov-layers:os-system-files` -- companion file overlay layer for bootc images
- `/ov-layers:bootc-base` -- bootc image base composition

## Related Commands
- `/ov:build` -- build the bootc image that consumes this layer
- `/ov:vm` -- boot the resulting bootc disk image

## When to Use This Skill

Use when the user asks about:

- Bootc OS-level configuration
- systemd preset-all for service enablement
- /opt directory permissions for rpm-ostree
- initramfs/dracut rebuild in bootc images
- Removing gaming-specific desktop entries

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
