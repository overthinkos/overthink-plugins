---
name: dbus
description: |
  D-Bus interaction inside containers via native Go godbus/dbus/v5.
  MUST be invoked before any work involving: ov dbus commands, desktop notifications, D-Bus method calls, service introspection, or session bus interaction.
---

# ov dbus -- D-Bus Interaction Inside Containers

## Overview

Send desktop notifications, call D-Bus methods, list services, and introspect objects inside running containers. Uses native Go (`godbus/dbus/v5`) -- no dependency on `dbus-send` CLI. All commands operate on the container's session bus.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Send notification | `ov dbus notify <image> "title" "body"` | Desktop notification via Notifications interface |
| Call method | `ov dbus call <image> <dest> <path> <method> [args...]` | Generic D-Bus method call |
| List services | `ov dbus list <image>` | List all registered session bus services |
| Introspect | `ov dbus introspect <image> <dest> <path>` | Introspect a service's interfaces and methods |

## Usage

### Desktop Notifications

```bash
# Simple notification
ov dbus notify sway-browser-vnc "Build Complete" "Image built successfully"

# Notification appears via swaync or other notification daemon
```

### Generic Method Calls

```bash
# Call any D-Bus method on the session bus
ov dbus call sway-browser-vnc org.freedesktop.Notifications /org/freedesktop/Notifications org.freedesktop.Notifications.GetCapabilities
```

### List Services

```bash
# See all registered services on the container's session bus
ov dbus list sway-browser-vnc
```

### Introspect a Service

```bash
# View interfaces, methods, signals, and properties
ov dbus introspect sway-browser-vnc org.freedesktop.Notifications /org/freedesktop/Notifications
```

## Prerequisites

- Container must have a D-Bus session bus running (provided by the `dbus` layer)
- For notifications: a notification daemon must be running (e.g., `swaync`)
- Container must be running (`ov start <image>`)

## Cross-References

- `/ov:cmd` -- single command execution in running containers
- `/ov:shell` -- interactive shell access
- `/ov-layers:dbus` -- D-Bus session bus layer configuration
- `/ov-layers:swaync` -- notification daemon layer
