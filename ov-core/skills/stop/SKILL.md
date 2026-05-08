---
name: stop
description: |
  Stop a running service container.
  MUST be invoked before any work involving: ov stop command, stopping containers, or halting services.
---

# ov stop -- Stop Service Container

## Overview

Stops a running service container. In quadlet mode, stops the systemd user service. In direct mode, stops the container via the container engine.

**Relationship to `ov deploy del`** — `ov stop <name>` is the ergonomic wrapper for `ov deploy del <name>` on a container deploy. The two are interchangeable for container deploys; `ov deploy del host` is required for the host-target teardown path (which runs ReverseOps, removes env.d files, and strips the shell managed block). See `/ov-core:deploy` and `/ov-advanced:local-deploy`.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Stop service | `ov stop <image>` | Stop the running container |
| With instance | `ov stop <image> -i 2` | Stop a specific instance |
| Stop + tear down FUSE | `ov stop <image> --unmount` | Stop the container AND unmount encrypted volumes (drops `ov-enc-<image>-<volume>.scope` units) |

## Usage

```bash
# Stop a running service
ov stop jupyter

# Stop a specific instance
ov stop ollama -i 2

# Stop and tear down encrypted FUSE mounts in one step
ov stop immich --unmount
```

## Flags

| Flag | Description |
|------|-------------|
| `-i, --instance INSTANCE` | Target a specific container instance |
| `--unmount` | After the container stop succeeds, also tear down `ov-enc-<image>-<volume>.scope` units via `encUnmount`. Best-effort: per-volume unmount failures emit a warning but don't propagate (the container has already stopped; the user can retry with `ov config unmount <image>`). Default `false` — plain `ov stop` leaves gocryptfs scopes running so they survive container restart (the original load-bearing design from `/ov-advanced:enc`). |

## Behavior

- **Quadlet mode:** Runs `systemctl --user stop ov-<image>.service`
- **Direct mode:** Runs `<engine> stop <container>`
- Does not remove the container or its configuration -- use `ov remove` for that
- Does not disable the service -- the container may restart on next login if enabled
- To stop and disable: `ov start <image> --enable=false` then `ov stop <image>`
- **By design**, plain `ov stop` does NOT tear down encrypted FUSE mounts — the `ov-enc-*.scope` units are deliberately decoupled from the container service cgroup so they survive `KillMode=mixed` on stop and let the next start fast-path through the `ov config mount` short-circuit. Use `--unmount` for the full teardown semantics.

## Cross-References

- `/ov-build:pull` -- Prerequisite: fetch the image into local storage; handles remote refs (`@github.com/...`) and the `ErrImageNotLocal` recovery path

- `/ov-core:start` -- Start services
- `/ov-core:remove` -- Remove containers, quadlets, and deploy config
- `/ov-core:status` -- Check service status
