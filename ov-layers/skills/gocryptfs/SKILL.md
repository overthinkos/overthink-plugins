---
name: gocryptfs
description: |
  Encrypted filesystem (gocryptfs) for ov config encrypted volume operations.
  Use when working with encrypted volumes, ov config mount/unmount, or filesystem encryption.
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

- `/ov-layers:virtualization` -- part of `ov-full` alongside gocryptfs
- `/ov-layers:socat` -- part of `ov-full` alongside gocryptfs

## When to Use This Skill

Use when the user asks about:

- Encrypted volumes or filesystems
- `ov config` encrypted volume operations (mount, unmount, status, passwd)
- The `gocryptfs` layer
