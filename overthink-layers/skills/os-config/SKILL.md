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
| Install files | `root.yml` |

## Usage

```yaml
# images.yml
my-bootc-image:
  bootc: true
  layers:
    - os-config
```

## Used In Images

- `/overthink-images:bazzite-ai` (disabled)

## When to Use This Skill

Use when the user asks about:

- Bootc OS-level configuration
- systemd preset-all for service enablement
- /opt directory permissions for rpm-ostree
- initramfs/dracut rebuild in bootc images
- Removing gaming-specific desktop entries
