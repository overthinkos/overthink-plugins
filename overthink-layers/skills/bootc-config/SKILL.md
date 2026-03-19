---
name: bootc-config
description: |
  Bootc system configuration: tty1 autologin, graphical target, pipewire/wireplumber enablement.
  Use when working with bootc images, autologin, or systemd graphical target setup.
---

# bootc-config -- bootc system configuration

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `root.yml`, `layer.yml` |

## Packages

- `sway-systemd` (RPM) -- Sway systemd integration

## Usage

```yaml
# images.yml -- typically used via bootc-base composition
my-bootc-image:
  base: "quay.io/fedora/fedora-bootc:43"
  bootc: true
  layers:
    - bootc-base
```

## Used In Images

Part of the `bootc-base` composition layer. Used transitively in bootc images.

## Related Layers

- `/overthink-layers:bootc-base` -- composition that includes this layer
- `/overthink-layers:sshd` -- SSH server (also in bootc-base)
- `/overthink-layers:qemu-guest-agent` -- QEMU agent (also in bootc-base)

## When to Use This Skill

Use when the user asks about:

- Bootc image autologin configuration
- systemd graphical target setup
- pipewire/wireplumber system service enablement
- tty1 autologin in bootable containers
