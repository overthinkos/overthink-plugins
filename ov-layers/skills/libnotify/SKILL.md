---
name: libnotify
description: |
  Desktop notification client library providing notify-send CLI.
  Use when working with notify-send, libnotify, or shell-based notifications.
---

# libnotify -- Desktop notification client

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `dbus` |

## Packages

- `libnotify` (RPM) -- provides `notify-send` CLI and `libnotify.so` shared library

## Usage

```yaml
# images.yml
my-image:
  layers:
    - libnotify
```

## Purpose

Provides `notify-send` as a convenience CLI for sending desktop notifications from shell scripts and manual use. The `ov dbus notify` command uses native Go D-Bus instead and does NOT depend on this layer.

### When to use `notify-send` vs `ov dbus notify`

| | `notify-send` | `ov dbus notify` |
|---|--------------|-----------------|
| Requires | `libnotify` layer | `ov` layer (or `gdbus` fallback) |
| Implementation | Shell command, libnotify C library | Native Go `godbus/dbus/v5` |
| Use case | Shell scripts inside container | Host-side automation, `ov cmd`/`ov tmux cmd` notifications |

## Related Layers

- `/ov-layers:dbus` -- D-Bus session bus (required dependency)
- `/ov-layers:swaync` -- notification daemon to display the notifications
- `/ov-layers:ov` -- alternative: native D-Bus via `ov dbus notify`

## When to Use This Skill

Use when the user asks about:

- `notify-send` inside containers
- Shell-based notification scripts
- The libnotify library or package
