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
# box.yml
my-os:
  layers:
    - bootc-base
```

## Related Layers

- `/charly-coder:sshd` -- SSH server (included)
- `/charly-distros:qemu-guest-agent` -- QEMU guest agent (included)
- `/charly-distros:bootc-config` -- bootc system configuration (included)
- `/charly-tools:charly` -- often paired for the full VM toolchain (charly binary + virtualization + gocryptfs + socat)

## Used In Images


## When to Use This Skill

Use when the user asks about:

- Bootable container (bootc) image setup
- VM base image composition
- OS-level image configuration
- SSH + QEMU guest agent setup

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
