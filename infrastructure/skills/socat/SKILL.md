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
| Install files | `charly.yml`, `task:` |

## Packages

RPM: `socat`, `iproute`

## Cross-distro coverage

`rpm:` (Fedora), `pac:` (Arch), `deb:` (Debian/Ubuntu — `socat`, `iproute2`). Full parity.

## Usage

```yaml
# charly.yml
my-image:
  candy:
    - socat
```

Typically not added directly. Auto-included when a layer uses `port_relay:` in its `charly.yml`.

**Note:** Chrome no longer uses socat for its DevTools port relay. Chrome now uses a `cdp-proxy` Python supervisord service (see `/charly-selkies:chrome`). Socat is still used by other `port_relay` consumers.

## Used In Images

- Part of the `charly` layer's full toolchain (used in `githubrunner`)

## Related Layers

- `/charly-infrastructure:virtualization` -- part of the `charly` layer alongside socat
- `/charly-infrastructure:gocryptfs` -- part of the `charly` layer alongside socat

## When to Use This Skill

Use when the user asks about:

- Port relay setup (eth0 to loopback)
- socat configuration
- Exposing loopback-only services
- The `socat` layer or `port_relay` field

## Author + Test References

- `/charly-image:layer` — layer authoring reference (tasks, vars, env_provide, tests block syntax)
- `/charly-eval:eval` — declarative testing framework for the `eval:` block
