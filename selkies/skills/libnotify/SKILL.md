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
# box.yml
my-image:
  layers:
    - libnotify
```

## Purpose

Provides `notify-send` as a convenience CLI for sending desktop notifications from shell scripts and manual use. The `ov eval dbus notify` command uses native Go D-Bus instead and does NOT depend on this layer.

### When to use `notify-send` vs `ov eval dbus notify`

| | `notify-send` | `ov eval dbus notify` |
|---|--------------|-----------------|
| Requires | `libnotify` layer | `ov` layer (or `gdbus` fallback) |
| Implementation | Shell command, libnotify C library | Native Go `godbus/dbus/v5` |
| Use case | Shell scripts inside container | Host-side automation, `ov cmd`/`ov tmux cmd` notifications |

## Related Layers

- `/ov-infrastructure:dbus-layer` -- D-Bus session bus (required dependency)
- `/ov-selkies:swaync` -- notification daemon to display the notifications
- `/ov-tools:ov` -- alternative: native D-Bus via `ov eval dbus notify`

## Used In Images

Not used in any current image definition. Optional notification CLI -- prefer `ov eval dbus notify` which uses native Go D-Bus.

## When to Use This Skill

Use when the user asks about:

- `notify-send` inside containers
- Shell-based notification scripts
- The libnotify library or package

## Related

- `/ov-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval box`, `ov eval live`)
