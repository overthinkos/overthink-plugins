---
name: dbus
description: |
  D-Bus session bus for inter-process communication inside containers.
  Use when working with D-Bus, desktop services, or Wayland compositor dependencies.
---

# dbus -- D-Bus session bus service

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord` |
| Service | `dbus` (supervisord, priority 2) |
| Install files | `root.yml` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `DBUS_SESSION_BUS_ADDRESS` | `unix:path=/tmp/dbus-session` |

## Packages

- `dbus-daemon` (RPM)

## Usage

```yaml
# images.yml -- typically not used directly; pulled in by sway
my-image:
  layers:
    - dbus
```

## Used In Images

Transitive dependency via `sway` in all desktop images.

## Related Layers

- `/overthink-layers:supervisord` -- process manager dependency
- `/overthink-layers:sway` -- primary consumer (Sway depends on dbus)

## When to Use This Skill

Use when the user asks about:

- D-Bus session bus configuration
- `DBUS_SESSION_BUS_ADDRESS` environment variable
- Inter-process communication in containers
- Desktop service dependencies
