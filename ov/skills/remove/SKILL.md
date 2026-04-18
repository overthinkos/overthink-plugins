---
name: remove
description: |
  Remove service container, quadlet file, and deploy.yml entry.
  MUST be invoked before any work involving: ov remove command, cleaning up containers, removing quadlets, or purging volumes.
---

# ov remove -- Remove Service Container

## Overview

Removes a service container along with its quadlet file and deploy.yml entry. Optionally purges named volumes and runs lifecycle hooks before removal.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Remove container | `ov remove <image>` | Remove container, quadlet, and deploy entry |
| Purge volumes | `ov remove <image> --purge` | Also remove named volumes |
| Keep deploy entry | `ov remove <image> --keep-deploy` | Keep deploy.yml entry for re-configuration |
| With env vars | `ov remove <image> -e KEY=VALUE` | Pass env vars to pre_remove hooks |

## Usage

```bash
# Remove a service (container + quadlet + deploy.yml entry)
ov remove jupyter

# Remove and purge all named volumes
ov remove ollama --purge

# Remove but keep deploy.yml entry for later re-configuration
ov remove immich --keep-deploy

# Pass environment variables to pre_remove hooks
ov remove openclaw -e CLEANUP_MODE=full
```

## Flags

| Flag | Description |
|------|-------------|
| `--purge` | Also remove named volumes associated with the image |
| `--keep-deploy` | Preserve the deploy.yml entry (useful for re-running `ov config`) |
| `-e, --env KEY=VALUE` | Pass environment variables to pre_remove lifecycle hooks |

## Behavior

1. Runs `pre_remove` lifecycle hooks (if defined in image labels)
2. Stops the container if still running
3. Removes the quadlet file (quadlet mode) or container (direct mode)
4. Removes the deploy.yml entry (unless `--keep-deploy`)
5. With `--purge`: removes named volumes (`ov-<image>-<name>`)

## Lifecycle Hooks

Images can define `pre_remove` hooks that run before the container is removed. These are useful for graceful shutdown, data export, or cleanup tasks. Use `-e` to pass context to hooks.

## Instance Removal

Remove specific instances with `-i`:

```bash
ov remove selkies-desktop -i 192.241.92.221          # Remove instance
ov remove selkies-desktop -i 192.241.92.221 --purge  # Also purge volumes
```

Instance removal automatically cleans the instance's `env_provides` and `mcp_provides` from `deploy.yml` provides section. Other instances of the same base image are unaffected.

## Cross-References

- `/ov:pull` -- Prerequisite: fetch the image into local storage; handles remote refs (`@github.com/...`) and the `ErrImageNotLocal` recovery path

- `/ov:stop` -- Stop without removing
- `/ov:config` -- Configure/reconfigure services (see "Full instance removal" for 3-step cleanup)
- `/ov:deploy` -- Deploy.yml management, tunnel configuration
- `/ov:service` -- Full service lifecycle
