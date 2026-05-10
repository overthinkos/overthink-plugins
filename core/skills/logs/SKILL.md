---
name: logs
description: |
  Service log viewing for running containers.
  MUST be invoked before any work involving: ov logs command, viewing container output, or debugging service issues.
---

# ov logs -- Service Log Viewing

## Overview

Displays logs from a running or recently stopped service container. In quadlet mode, reads from journalctl. In direct mode, reads from the container engine's log store.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| View logs | `ov logs <image>` | Show recent logs |
| Follow logs | `ov logs <image> -f` | Stream logs in real time |
| Tail N lines | `ov logs <image> -n 50` | Show last N lines |
| With instance | `ov logs <image> -i 2` | Logs from specific instance |

## Usage

```bash
# View recent logs
ov logs jupyter

# Follow logs in real time (Ctrl+C to stop)
ov logs ollama -f

# Show last 100 lines
ov logs immich -n 100

# Logs from a specific instance
ov logs openclaw -i 2

# Combine follow with tail
ov logs comfyui -f -n 50
```

## Flags

| Flag | Description |
|------|-------------|
| `-f, --follow` | Stream logs continuously |
| `-n, --lines LINES` | Number of recent lines to show |
| `-i, --instance INSTANCE` | Target a specific container instance |

## Behavior

- **Quadlet mode:** Uses `journalctl --user -u ov-<image>.service`
- **Direct mode:** Uses `<engine> logs <container>`
- Quadlet mode is the default and recommended deployment method
- Logs include both stdout and stderr from the container

## Cross-References

- `/ov-build:pull` -- Prerequisite: fetch the image into local storage; handles remote refs (`@github.com/...`) and the `ErrImageNotLocal` recovery path

- `/ov-core:status` -- Check service status
- `/ov-core:service` -- Service lifecycle management
- `/ov-core:start` -- Starting services
