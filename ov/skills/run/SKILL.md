---
name: run
description: |
  Runtime operations redirect. Content has been split into dedicated skills.
  See shell, service, cdp, alias, and config skills.
---

# Run - Runtime Operations

Runtime operations are documented in dedicated skills:

| Topic | Skill | Covers |
|-------|-------|--------|
| Shell & execution | `/ov:shell` | `ov shell`, `--tty`, `-c` commands, exec into running containers, port relay, device auto-detection, env vars, remote refs |
| Service management | `/ov:service` | `ov start/stop/enable/disable/status/logs/update/remove`, supervisord services, lifecycle hooks, multi-instance, seed |
| CDP (Chrome DevTools) | `/ov:cdp` | `ov cdp` commands, CDP, Chrome DevTools, OAuth flows |
| Sway compositor | `/ov:sway` | `ov sway` commands, window management, workspaces, outputs |
| Aliases | `/ov:alias` | `ov alias add/remove/list/install/uninstall` |
| Configuration | `/ov:config` | `ov config get/set/list/reset/path`, engine settings, bind address |
