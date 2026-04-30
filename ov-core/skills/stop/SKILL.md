---
name: stop
description: |
  Stop a running service container.
  MUST be invoked before any work involving: ov stop command, stopping containers, or halting services.
---

# ov stop -- Stop Service Container

## Overview

Stops a running service container. In quadlet mode, stops the systemd user service. In direct mode, stops the container via the container engine.

**Relationship to `ov deploy del`** — `ov stop <name>` is the ergonomic wrapper for `ov deploy del <name>` on a container deploy. The two are interchangeable for container deploys; `ov deploy del host` is required for the host-target teardown path (which runs ReverseOps, removes env.d files, and strips the shell managed block). See `/ov-core:deploy` and `/ov-advanced:host-deploy`.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Stop service | `ov stop <image>` | Stop the running container |
| With instance | `ov stop <image> -i 2` | Stop a specific instance |

## Usage

```bash
# Stop a running service
ov stop jupyter

# Stop a specific instance
ov stop ollama -i 2
```

## Flags

| Flag | Description |
|------|-------------|
| `-i, --instance INSTANCE` | Target a specific container instance |

## Behavior

- **Quadlet mode:** Runs `systemctl --user stop ov-<image>.service`
- **Direct mode:** Runs `<engine> stop <container>`
- Does not remove the container or its configuration -- use `ov remove` for that
- Does not disable the service -- the container may restart on next login if enabled
- To stop and disable: `ov start <image> --enable=false` then `ov stop <image>`

## Cross-References

- `/ov-build:pull` -- Prerequisite: fetch the image into local storage; handles remote refs (`@github.com/...`) and the `ErrImageNotLocal` recovery path

- `/ov-core:start` -- Start services
- `/ov-core:remove` -- Remove containers, quadlets, and deploy config
- `/ov-core:status` -- Check service status
