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
| Install files | `task:` |

## Usage

```yaml
# box.yml
my-bootc-image:
  bootc: true
  layers:
    - os-config
```

## Used In Images

- `/charly-distros:bazzite`

## Related Layers
- `/charly-distros:os-system-files` -- companion file overlay layer for bootc images
- `/charly-distros:bootc-base` -- bootc image base composition

## Related Commands
- `/charly-build:build` -- build the bootc image that consumes this layer
- `/charly-vm:vm` -- boot the resulting bootc disk image

## When to Use This Skill

Use when the user asks about:

- Bootc OS-level configuration
- systemd preset-all for service enablement
- /opt directory permissions for rpm-ostree
- initramfs/dracut rebuild in bootc images
- Removing gaming-specific desktop entries

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
