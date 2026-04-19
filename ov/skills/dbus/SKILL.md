---
name: dbus
description: |
  D-Bus interaction inside containers via native Go godbus/dbus/v5.
  MUST be invoked before any work involving: ov test dbus commands, desktop notifications, D-Bus method calls, service introspection, or session bus interaction.
---

# ov test dbus -- D-Bus Interaction Inside Containers

## Overview

Send desktop notifications, call D-Bus methods, list services, and introspect objects inside running containers. Uses native Go (`godbus/dbus/v5`) -- no dependency on `dbus-send` CLI. All commands operate on the container's session bus.

### Also as a declarative verb

Every `ov test dbus <method>` (list/call/introspect/notify) is authorable as a `dbus:` verb inside a `tests:` block. Method-specific fields (`dest:`, `path:`, `method:`, `args:`, `text:`) are siblings of the verb line. See `/ov:test` for the full YAML shape. Example: `- dbus: list\n  stdout:\n    contains: "org.freedesktop.Notifications"`.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Send notification | `ov test dbus notify <image> "title" "body"` | Desktop notification via Notifications interface |
| Call method | `ov test dbus call <image> <dest> <path> <method> [args...]` | Generic D-Bus method call |
| List services | `ov test dbus list <image>` | List all registered session bus services |
| Introspect | `ov test dbus introspect <image> <dest> <path>` | Introspect a service's interfaces and methods |

## Usage

### Desktop Notifications

```bash
# Simple notification
ov test dbus notify sway-browser-vnc "Build Complete" "Image built successfully"

# Notification appears via swaync or other notification daemon
```

### Generic Method Calls

```bash
# Call any D-Bus method on the session bus
ov test dbus call sway-browser-vnc org.freedesktop.Notifications /org/freedesktop/Notifications org.freedesktop.Notifications.GetCapabilities
```

### List Services

```bash
# See all registered services on the container's session bus
ov test dbus list sway-browser-vnc
```

### Introspect a Service

```bash
# View interfaces, methods, signals, and properties
ov test dbus introspect sway-browser-vnc org.freedesktop.Notifications /org/freedesktop/Notifications
```

## Prerequisites

- Container must have a D-Bus session bus running (provided by the `dbus` layer)
- For notifications: a notification daemon must be running (e.g., `swaync`)
- Container must be running (`ov start <image>`)

## Cross-References

- `/ov:test` -- parent router; `ov test dbus …` is how every invocation is dispatched.
- `/ov:cdp` -- Chrome DevTools Protocol (sibling verb under `ov test`).
- `/ov:wl` -- Wayland desktop automation (sibling verb under `ov test`).
- `/ov:vnc` -- VNC desktop automation (sibling verb under `ov test`).
- `/ov:cmd` -- single command execution in running containers
- `/ov:shell` -- interactive shell access
- `/ov-layers:dbus` -- D-Bus session bus layer configuration
- `/ov-layers:swaync` -- notification daemon layer
