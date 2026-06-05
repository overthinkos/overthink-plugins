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
| Install files | `candy.yml` (packages only) |

## Packages

RPM: `gocryptfs`

## Cross-distro coverage

`rpm:` (Fedora), `pac:` (Arch — community), `deb:` (Debian/Ubuntu — `gocryptfs` available in Debian main) — full parity.

## Usage

```yaml
# box.yml
my-image:
  layers:
    - gocryptfs
```

Typically used as part of the `ov` layer (the full toolchain: ov binary + virtualization + gocryptfs + socat) rather than directly.

## Runtime Behavior

When `ov config mount` or `ov start` mounts encrypted volumes, each gocryptfs daemon runs inside a `systemd-run --scope --user --unit=ov-enc-<image>-<volume>` scope unit. This decouples the FUSE mount lifecycle from the container service — mounts survive container stop/restart and remain browsable on the host.

The `-allow_other` flag is always passed to gocryptfs (required for rootless podman with `--userns=keep-id`). gocryptfs auto-enables `default_permissions`, so kernel UNIX permission checks still apply.

See `/ov-automation:enc` for full encrypted volume operations documentation.

## Used In Images

- Part of the `ov` layer's full toolchain (used in `githubrunner`)

## Related Layers

- `/ov-infrastructure:virtualization` -- part of the `ov` layer alongside gocryptfs
- `/ov-infrastructure:socat` -- part of the `ov` layer alongside gocryptfs

## When to Use This Skill

Use when the user asks about:

- Encrypted volumes or filesystems
- `ov config` encrypted volume operations (mount, unmount, status, passwd)
- The `gocryptfs` layer
- systemd scope units for encrypted mounts (`ov-enc-*`)

## Author + Test References

- `/ov-image:layer` — layer authoring reference (tasks, vars, env_provide, tests block syntax)
- `/ov-eval:eval` — declarative testing framework for the `eval:` block
