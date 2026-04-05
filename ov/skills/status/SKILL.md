---
name: status
description: |
  Service status display with tool probes and device detection.
  MUST be invoked before any work involving: ov status command, checking container state, tool availability, port mapping, or JSON status output.
---

# ov status -- Service Status Display

## Overview

Display running container status in table or detail mode. Includes concurrent tool probes (CDP, VNC, Sway, Wayland) with 2-second timeouts, GPU device detection, and port mapping.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Table (running) | `ov status` | Show all running services |
| Table (all) | `ov status --all` | Include stopped and enabled services |
| Detail | `ov status <image>` | Key-value detail for one service |
| JSON output | `ov status --json` | Machine-readable JSON |

## Table Mode Columns

| Column | Description |
|--------|-------------|
| IMAGE | Image name |
| STATUS | running, stopped, enabled, etc. |
| PORTS | Mapped host:container ports |
| DEVICES | Detected GPU devices (nvidia, amd) |
| TOOLS | Available tools with ports or socket names |

## Detail Mode Fields

`ov status <image>` shows:

| Field | Example |
|-------|---------|
| Image | `ghcr.io/overthinkos/jupyter:latest` |
| Status | `running` |
| Container | `ov-jupyter` |
| Mode | `quadlet` |
| Ports | `8888/tcp -> 127.0.0.1:8888` |
| Devices | `nvidia (CUDA)` |
| Tools | `cdp:9222, vnc:5900, sway, wl` |
| Volumes | `data: bind /home/user/data` |
| Workspace | `/home/user/notebooks` |
| Network | `host` |
| Tunnel | `cloudflare: jupyter.example.com` |

## Tool Probes

Tools are detected via concurrent 2-second timeout probes:

| Tool | Probe Method | Display |
|------|-------------|---------|
| cdp | HTTP request to CDP port | `cdp:9222` |
| vnc | RFB protocol handshake | `vnc:5900` |
| sway | IPC socket check | `sway` |
| wl | Command detection | `wl` |

Port-based tools show `name:port`. Socket-based tools show just the name.

## Usage

```bash
# Quick overview of all running services
ov status

# Include stopped services
ov status --all

# Detailed info for one service
ov status jupyter

# JSON for scripting
ov status --json | jq '.[] | select(.status == "running")'
```

## Cross-References

- `/ov:start` -- start a service
- `/ov:stop` -- stop a service (via `/ov:service`)
- `/ov:logs` -- view service logs (via `/ov:service`)
- `/ov:service` -- full service lifecycle management
