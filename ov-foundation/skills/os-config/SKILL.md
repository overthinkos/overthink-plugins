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

- `/ov-foundation:bazzite-ai` (disabled)

## Related Layers
- `/ov-foundation:os-system-files` -- companion file overlay layer for bootc images
- `/ov-foundation:bootc-base` -- bootc image base composition

## Related Commands
- `/ov-build:build` -- build the bootc image that consumes this layer
- `/ov-advanced:vm` -- boot the resulting bootc disk image

## When to Use This Skill

Use when the user asks about:

- Bootc OS-level configuration
- systemd preset-all for service enablement
- /opt directory permissions for rpm-ostree
- initramfs/dracut rebuild in bootc images
- Removing gaming-specific desktop entries

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
