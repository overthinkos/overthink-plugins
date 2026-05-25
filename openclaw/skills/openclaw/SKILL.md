---
name: openclaw
description: |
  Headless OpenClaw AI gateway image. Runs the gateway on port 18789
  without a desktop environment. Use when working with the headless
  MUST be invoked before building, deploying, configuring, or troubleshooting the openclaw image.
---

# openclaw

Headless OpenClaw AI gateway — no desktop, no browser, just the gateway service.

## Image Properties

| Property | Value |
|----------|-------|
| Base | cachyos |
| Layers | agent-forwarding, openclaw, dbus, ov |
| Platforms | linux/amd64 |
| Ports | 18789 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `cachyos` (docker.io/cachyos/cachyos-v3, digest-pinned, Arch-derived)
2. `pixi` → `python` → `supervisord` (transitive)
3. `nodejs` (transitive via openclaw)
4. `openclaw` — gateway on :18789, data volume
5. `dbus` — message bus; `ov` — ov CLI toolchain

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway + Control UI | HTTP |

## Quick Start

```bash
ov image build openclaw
ov config openclaw
ov start openclaw
# Gateway at http://localhost:18789
```

## Key Layers

- `/ov-openclaw:openclaw` — gateway npm package, supervisord service, data volume

## Related Images

- `/ov-openclaw:openclaw-full` — maximal variant (gateway + browser + all tools)

## Verification

After `ov start`:
- `ov status openclaw` — container running
- `ov service status openclaw` — all services RUNNING
- `curl -s http://localhost:18789` — OpenClaw gateway responds

## Port Relay Architecture

OpenClaw gateway (18789) uses port relay (socat) — the gateway binds to loopback, socat forwards from the container interface. This avoids the `allowedOrigins` requirement for the Control UI.

## When to Use This Skill

**MUST be invoked** when the task involves the headless openclaw image, deploying openclaw without a desktop, or comparing openclaw variants. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov-image:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
