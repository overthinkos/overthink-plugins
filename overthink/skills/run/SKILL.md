---
name: run
description: |
  Runtime operations redirect. Content has been split into dedicated skills.
  See shell, service, browser, alias, and config skills.
---

# Run - Runtime Operations

Runtime operations are documented in dedicated skills:

| Topic | Skill | Covers |
|-------|-------|--------|
| Shell & execution | `/overthink:shell` | `ov shell`, `--tty`, `-c` commands, exec into running containers, port relay, device auto-detection, env vars, remote refs |
| Service management | `/overthink:service` | `ov start/stop/enable/disable/status/logs/update/remove`, supervisord services, lifecycle hooks, multi-instance, seed |
| Browser automation | `/overthink:browser` | `ov browser` commands, CDP, Chrome DevTools, OAuth flows |
| Aliases | `/overthink:alias` | `ov alias add/remove/list/install/uninstall` |
| Configuration | `/overthink:config` | `ov config get/set/list/reset/path`, engine settings, bind address |
