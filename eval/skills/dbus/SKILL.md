---
name: dbus
description: |
  D-Bus interaction inside containers via native Go godbus/dbus/v5.
  MUST be invoked before any work involving: charly eval dbus commands, desktop notifications, D-Bus method calls, service introspection, or session bus interaction.
---

# charly eval dbus -- D-Bus Interaction Inside Containers

## Overview

Send desktop notifications, call D-Bus methods, list services, and introspect objects inside running containers. Uses native Go (`godbus/dbus/v5`) -- no dependency on `dbus-send` CLI. All commands operate on the container's session bus.

### Also as a declarative verb

Every `charly eval dbus <method>` (list/call/introspect/notify) is authorable as a `dbus:` verb inside a `eval:` block. Method-specific fields (`dest:`, `path:`, `method:`, `args:`, `text:`) are siblings of the verb line. See `/charly-eval:eval` for the full YAML shape. Example: `- dbus: list\n  stdout:\n    contains: "org.freedesktop.Notifications"`.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Send notification | `charly eval dbus notify <image> "title" "body"` | Desktop notification via Notifications interface |
| Call method | `charly eval dbus call <image> <dest> <path> <method> [args...]` | Generic D-Bus method call |
| List services | `charly eval dbus list <image>` | List all registered session bus services |
| Introspect | `charly eval dbus introspect <image> <dest> <path>` | Introspect a service's interfaces and methods |

## Usage

### Desktop Notifications

```bash
# Simple notification
charly eval dbus notify sway-browser-vnc "Build Complete" "Image built successfully"

# Notification appears via swaync or other notification daemon
```

### Generic Method Calls

```bash
# Call any D-Bus method on the session bus
charly eval dbus call sway-browser-vnc org.freedesktop.Notifications /org/freedesktop/Notifications org.freedesktop.Notifications.GetCapabilities
```

### List Services

```bash
# See all registered services on the container's session bus
charly eval dbus list sway-browser-vnc
```

### Introspect a Service

```bash
# View interfaces, methods, signals, and properties
charly eval dbus introspect sway-browser-vnc org.freedesktop.Notifications /org/freedesktop/Notifications
```

## Prerequisites

- Container must have a D-Bus session bus running (provided by the `dbus` layer)
- For notifications: a notification daemon must be running (e.g., `swaync`)
- Container must be running (`charly start <image>`)

## Cross-References

- `/charly-eval:eval` -- parent router; `charly eval dbus …` is how every invocation is dispatched.
- `/charly-eval:cdp` -- Chrome DevTools Protocol (sibling verb under `charly eval`).
- `/charly-eval:wl` -- Wayland desktop automation (sibling verb under `charly eval`).
- `/charly-eval:vnc` -- VNC desktop automation (sibling verb under `charly eval`).
- `/charly-core:cmd` -- single command execution in running containers
- `/charly-core:shell` -- interactive shell access
- `/charly-infrastructure:dbus-layer` -- D-Bus session bus layer configuration
- `/charly-selkies:swaync` -- notification daemon layer
