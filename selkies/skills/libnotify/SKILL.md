---
name: libnotify
description: |
  Desktop notification client library providing notify-send CLI.
  Use when working with notify-send, libnotify, or shell-based notifications.
---

# libnotify -- Desktop notification client

## Candy Properties

| Property | Value |
|----------|-------|
| Dependencies | `dbus` |

## Packages

- `libnotify` (RPM) -- provides `notify-send` CLI and `libnotify.so` shared library

## Usage

```yaml
# charly.yml
my-image:
  candy:
    - libnotify
```

## Purpose

Provides `notify-send` as a convenience CLI for sending desktop notifications from shell scripts and manual use. The `charly check dbus notify` command uses native Go D-Bus instead and does NOT depend on this candy.

### When to use `notify-send` vs `charly check dbus notify`

| | `notify-send` | `charly check dbus notify` |
|---|--------------|-----------------|
| Requires | `libnotify` candy | `charly` candy (or `gdbus` fallback) |
| Implementation | Shell command, libnotify C library | Native Go `godbus/dbus/v5` |
| Use case | Shell scripts inside container | Host-side automation, `charly cmd`/`charly tmux cmd` notifications |

## Related Candies

- `/charly-infrastructure:dbus-layer` -- D-Bus session bus (required dependency)
- `/charly-selkies:swaync` -- notification daemon to display the notifications
- `/charly-tools:charly` -- alternative: native D-Bus via `charly check dbus notify`

## Used In Boxes

Not used in any current box definition. Optional notification CLI -- prefer `charly check dbus notify` which uses native Go D-Bus.

## When to Use This Skill

Use when the user asks about:

- `notify-send` inside containers
- Shell-based notification scripts
- The libnotify library or package

## Related

- `/charly-image:layer` — candy authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-check:check` — declarative testing (`check:` block, `charly check box`, `charly check live`)
