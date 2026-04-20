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

## Cross-distro coverage

`rpm:` (Fedora), `pac:` (Arch — community), `deb:` (Debian/Ubuntu — `gocryptfs` available in Debian main) — full parity.

## Usage

```yaml
# image.yml
my-image:
  layers:
    - gocryptfs
```

Typically used as part of the `ov-full` composition layer rather than directly.

## Runtime Behavior

When `ov config mount` or `ov start` mounts encrypted volumes, each gocryptfs daemon runs inside a `systemd-run --scope --user --unit=ov-enc-<image>-<volume>` scope unit. This decouples the FUSE mount lifecycle from the container service — mounts survive container stop/restart and remain browsable on the host.

The `-allow_other` flag is always passed to gocryptfs (required for rootless podman with `--userns=keep-id`). gocryptfs auto-enables `default_permissions`, so kernel UNIX permission checks still apply.

See `/ov:enc` for full encrypted volume operations documentation.

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
- systemd scope units for encrypted mounts (`ov-enc-*`)

## Author + Test References

- `/ov:layer` — layer authoring reference (tasks, vars, env_provides, tests block syntax)
- `/ov:test` — declarative testing framework for the `tests:` block
