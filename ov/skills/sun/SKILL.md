---
name: sun
description: |
  Sunshine game streaming management: ov sun commands for credential setup,
  Moonlight pairing, client management, config, and service control.
  Use when working with Sunshine inside containers.
---

# Sun - Sunshine Game Streaming Management

## Overview

`ov sun` commands manage Sunshine game streaming servers inside running containers via the Sunshine REST API (HTTPS on port 47990). Provides credential setup, Moonlight client pairing, configuration management, and service control.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Status | `ov sun status <image>` | Check Sunshine health, show version/encoder |
| Set password | `ov sun passwd <image>` | Set Web UI credentials (stores in ov config) |
| Pair client | `ov sun pair <image> <pin>` | Submit 4-digit PIN from Moonlight client |
| List clients | `ov sun clients <image>` | List paired Moonlight clients |
| Unpair | `ov sun unpair <image> [uuid]` | Unpair one or all clients |
| List apps | `ov sun apps <image>` | List GameStream applications |
| Show config | `ov sun config <image>` | Show current Sunshine configuration |
| Set config | `ov sun set <image> <key> <value>` | Modify a config value |
| Restart | `ov sun restart <image>` | Restart Sunshine service |
| Web UI URL | `ov sun url <image>` | Print Sunshine Web UI URL |

## Architecture

```
CLI command -> resolveSunContainer (engine + container name)
           -> resolveSunAddress (docker/podman port <name> 47990)
           -> resolveSunCredentials (ov config + SUNSHINE_USER/PASSWORD env)
           -> NewSunshineClient(baseURL, user, pass) -> HTTPS REST API
```

Pure API implementation — all operations use Sunshine's REST API (HTTPS, Basic Auth, self-signed cert). No `docker exec` needed. The only exception is `ov sun restart` which delegates to `ServiceRestartCmd` (supervisorctl via ov code paths).

## Requirements

- Container must include the `sunshine` layer (port 47990)
- Container must be running (`ov start` or `ov enable`)
- Credentials must be set via `ov sun passwd` before most commands work

## Commands

### Status
```bash
ov sun status sway-browser-sunshine
# Output:
# Container: ov-sway-browser-sunshine
# sunshine                         RUNNING   pid 42, uptime 0:05:00
# Version:   2026.321.25052
# Encoder:   nvenc
# Capture:   wlroots
# Platform:  linux
```

### Set Password
```bash
ov sun passwd sway-browser-sunshine --generate   # auto-generate, print to stdout
ov sun passwd sway-browser-sunshine              # prompt for password
ov sun passwd sway-browser-sunshine --user admin --generate  # custom username
```

Sets Sunshine Web UI credentials via `POST /api/password`. On first run (no credentials set), the API accepts the request without authentication. Stores username and password in `ov config`.

### Pair Client
```bash
ov sun pair sway-browser-sunshine 1234                          # submit PIN
ov sun pair sway-browser-sunshine 5678 --name "My Moonlight PC" # with client name
```

**Pairing flow:**
1. Open Moonlight client, select "Add Host", enter container host IP
2. Moonlight displays a 4-digit PIN
3. Run `ov sun pair <image> <pin>` to complete pairing
4. Moonlight connects and streaming begins

### List Clients
```bash
ov sun clients sway-browser-sunshine
# Output:
# abc-123-def   Moonlight PC
# ghi-456-jkl   Phone
```

### Unpair
```bash
ov sun unpair sway-browser-sunshine abc-123-def   # unpair specific client
ov sun unpair sway-browser-sunshine --all          # unpair all clients
```

### List Applications
```bash
ov sun apps sway-browser-sunshine
# Output:
# 0   Desktop   (desktop)

ov sun apps sway-browser-sunshine-steam
# Output:
# 0   Desktop   (desktop)
# 1   Steam Big Picture   (desktop)
```

Images with the `/ov-layers:steam` layer have "Steam Big Picture" pre-configured in `apps.json`.

### Show Config
```bash
ov sun config sway-browser-sunshine              # key=value format
ov sun config sway-browser-sunshine --json       # raw JSON
```

### Set Config
```bash
ov sun set sway-browser-sunshine encoder nvenc
ov sun set sway-browser-sunshine min_log_level info
ov sun set sway-browser-sunshine capture wlroots
```

### Restart
```bash
ov sun restart sway-browser-sunshine
```

Delegates to `ov service restart <image> sunshine` via ov's `ServiceRestartCmd`.

### Web UI URL
```bash
ov sun url sway-browser-sunshine
# Output: https://127.0.0.1:47990
```

## Credential Storage

Stored in `~/.config/ov/config.yml`:

```yaml
sunshine_users:
  sway-browser-sunshine: sunshine
sunshine_passwords:
  sway-browser-sunshine: <password>
```

**Resolution chain** (per command):
1. `SUNSHINE_USER` / `SUNSHINE_PASSWORD` env vars (CI/automation)
2. `sunshine.user.<image>-<instance>` + `sunshine.password.<image>-<instance>` (instance-specific)
3. `sunshine.user.<image>` + `sunshine.password.<image>` (image-level)
4. Error — credentials required for API access

```bash
# Set credentials programmatically
ov config set sunshine.user.sway-browser-sunshine admin
ov config set sunshine.password.sway-browser-sunshine mysecret

# One-off env var override
SUNSHINE_USER=admin SUNSHINE_PASSWORD=secret ov sun status sway-browser-sunshine
```

## Sunshine Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 47984 | TCP | GameStream HTTPS API |
| 47989 | TCP/HTTP | GameStream HTTP API |
| 47990 | TCP/HTTPS | Web UI + REST API |
| 48010 | TCP | RTSP session setup |
| 47998 | UDP | Video stream (NVENC) |
| 47999 | UDP | Control channel |
| 48000 | UDP | Audio stream (Opus) |

## Cross-References

- `/ov:moon` — GameStream client protocol (client-side pairing, app launch/quit)
- `/ov-layers:sunshine` — Sunshine layer properties (ports, deps, service config)
- `/ov-layers:sway-desktop-sunshine` — GPU-accelerated desktop composition with Sunshine
- `/ov:vnc` — VNC automation (alternative remote access, wayvnc)
- `/ov:cdp` — Chrome DevTools Protocol (browser automation, same container)
- `/ov:service` — Supervisord service management
- `/ov:config` — Sunshine credential storage (`sunshine.user.<image>`, `sunshine.password.<image>`)

## When to Use This Skill

**MUST be invoked** when the task involves Sunshine management, Moonlight pairing, Sunshine credentials, Sunshine configuration, or `ov sun` commands. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Sunshine management. Use for server setup and client pairing. For desktop interaction, use `/ov:cdp` (DOM), `/ov:wl` (Wayland), or `/ov:vnc` (VNC).
