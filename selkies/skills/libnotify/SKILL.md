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

Provides `notify-send` as a convenience CLI for sending desktop notifications from shell scripts and manual use. The `charly eval dbus notify` command uses native Go D-Bus instead and does NOT depend on this layer.

### When to use `notify-send` vs `charly eval dbus notify`

| | `notify-send` | `charly eval dbus notify` |
|---|--------------|-----------------|
| Requires | `libnotify` layer | `charly` layer (or `gdbus` fallback) |
| Implementation | Shell command, libnotify C library | Native Go `godbus/dbus/v5` |
| Use case | Shell scripts inside container | Host-side automation, `charly cmd`/`charly tmux cmd` notifications |

## Related Layers

- `/charly-infrastructure:dbus-layer` -- D-Bus session bus (required dependency)
- `/charly-selkies:swaync` -- notification daemon to display the notifications
- `/charly-tools:charly` -- alternative: native D-Bus via `charly eval dbus notify`

## Used In Images

Not used in any current image definition. Optional notification CLI -- prefer `charly eval dbus notify` which uses native Go D-Bus.

## When to Use This Skill

Use when the user asks about:

- `notify-send` inside containers
- Shell-based notification scripts
- The libnotify library or package

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
