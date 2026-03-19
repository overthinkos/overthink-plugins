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
| Install files | `layer.yml`, `root.yml` |

## Packages

RPM: `socat`, `iproute`

## Usage

```yaml
# images.yml
my-image:
  layers:
    - socat
```

Typically not added directly. Auto-included when a layer uses `port_relay:` in its `layer.yml`.

## Used In Images

- Part of `ov-full` composition layer (used in `githubrunner`)
- Transitive dependency of `chrome` (via `port_relay` on port 9222)

## Related Layers

- `/ov-layers:chrome` -- depends on socat for DevTools port relay
- `/ov-layers:virtualization` -- part of `ov-full` alongside socat
- `/ov-layers:gocryptfs` -- part of `ov-full` alongside socat

## When to Use This Skill

Use when the user asks about:

- Port relay setup (eth0 to loopback)
- socat configuration
- Exposing loopback-only services
- The `socat` layer or `port_relay` field
