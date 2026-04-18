---
name: socat
description: |
  Socket relay tool for VM console access and port relays (eth0 to loopback).
  Use when working with port relays, socat, or loopback service exposure.
---

# socat -- Socket relay tool

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `tasks:` |

## Packages

RPM: `socat`, `iproute`

## Usage

```yaml
# image.yml
my-image:
  layers:
    - socat
```

Typically not added directly. Auto-included when a layer uses `port_relay:` in its `layer.yml`.

**Note:** Chrome no longer uses socat for its DevTools port relay. Chrome now uses a `cdp-proxy` Python supervisord service (see `/ov-layers:chrome`). Socat is still used by other `port_relay` consumers.

## Used In Images

- Part of `ov-full` composition layer (used in `githubrunner`)

## Related Layers

- `/ov-layers:virtualization` -- part of `ov-full` alongside socat
- `/ov-layers:gocryptfs` -- part of `ov-full` alongside socat

## When to Use This Skill

Use when the user asks about:

- Port relay setup (eth0 to loopback)
- socat configuration
- Exposing loopback-only services
- The `socat` layer or `port_relay` field

## Author + Test References

- `/ov:layer` — layer authoring reference (tasks, vars, env_provides, tests block syntax)
- `/ov:test` — declarative testing framework for the `tests:` block
