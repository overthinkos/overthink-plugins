---
name: gocryptfs
description: |
  Encrypted filesystem (gocryptfs) for ov enc operations.
  Use when working with encrypted volumes, ov enc, or filesystem encryption.
---

# gocryptfs -- Encrypted filesystem support

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |

## Packages

RPM: `gocryptfs`

## Usage

```yaml
# images.yml
my-image:
  layers:
    - gocryptfs
```

Typically used as part of the `ov-full` composition layer rather than directly.

## Used In Images

- Part of `ov-full` composition layer (used in `githubrunner`)

## Related Layers

- `/overthink-layers:virtualization` -- part of `ov-full` alongside gocryptfs
- `/overthink-layers:socat` -- part of `ov-full` alongside gocryptfs

## When to Use This Skill

Use when the user asks about:

- Encrypted volumes or filesystems
- `ov enc` operations (init, mount, unmount)
- The `gocryptfs` layer
