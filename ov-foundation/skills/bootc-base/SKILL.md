---
name: bootc-base
description: |
  Base composition for bootc OS images including SSH, QEMU guest agent, and bootc config.
  Use when working with bootable container images, VMs, or OS-level configuration.
---

# bootc-base -- Base composition for bootable container images

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `sshd`, `qemu-guest-agent`, `bootc-config` |
| Install files | none (pure composition) |

## Usage

```yaml
# image.yml
my-os:
  layers:
    - bootc-base
```

## Related Layers

- `/ov-coder:sshd` -- SSH server (included)
- `/ov-foundation:qemu-guest-agent` -- QEMU guest agent (included)
- `/ov-foundation:bootc-config` -- bootc system configuration (included)
- `/ov-coder:ov-full` -- often paired for full VM toolchain

## Used In Images

- `/ov-openclaw:openclaw-browser-bootc` (disabled image)

## When to Use This Skill

Use when the user asks about:

- Bootable container (bootc) image setup
- VM base image composition
- OS-level image configuration
- SSH + QEMU guest agent setup

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
