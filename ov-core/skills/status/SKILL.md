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

## Known display issue: `Volumes:` reads from image labels

The `Volumes:` field is populated from the image's OCI labels (`ExtractMetadata` in `ov/status.go:686–698`), not from the live container's actual mounts. This means a volume deployed with `--bind <name>=<path>` will appear in `ov status` as `ov-<image>-<volume> -> /container/path` (the image default from the OCI label), not as `<host-path> -> /container/path` (the actual runtime bind mount).

The running container is functionally correct — only the display is misleading. Authoritative sources for the live volume backing:

```bash
# What the live container is actually mounting
podman inspect <container> --format '{{range .Mounts}}{{.Type}}:{{.Source}}->{{.Destination}} {{"\n"}}{{end}}'

# The deploy.yml record (what ov config resolved)
ov deploy show <image>
```

Use either of the above when you need to confirm that a `--bind` / `--encrypt` override actually took effect.

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

- `/ov-build:pull` -- Prerequisite: fetch the image into local storage; handles remote refs (`@github.com/...`) and the `ErrImageNotLocal` recovery path

- `/ov-core:start` -- start a service
- `/ov-core:stop` -- stop a service (via `/ov-core:service`)
- `/ov-core:logs` -- view service logs (via `/ov-core:service`)
- `/ov-core:service` -- full service lifecycle management

## Live-deploy verification is mandatory (see `/ov-build:eval` 10 standards)

Changes that touch this verb's output must reach a healthy deployment on a target explicitly marked `disposable: true` (see `/ov-dev:disposable`). Use `ov rebuild <name>` to destroy + rebuild unattended on any disposable target. Never experiment on a non-disposable deploy — set up a disposable one first with `ov deploy add <name> <ref> --disposable` or mark a VM in vms.yml.

**After committing the source-level fix, `ov rebuild` the disposable target ONCE MORE from clean and re-run the full verification.** A fix that passes only on a hand-patched target is not a real fix — it's a regression waiting for the next unrelated rebuild. Paste BOTH the exploratory-pass output and the fresh-rebuild-pass output into the conversation.

Unit tests + a clean compile are necessary but not sufficient. See CLAUDE.md R1–R10.
