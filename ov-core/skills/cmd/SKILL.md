---
name: cmd
description: |
  Single command execution in a running container with D-Bus notification.
  MUST be invoked before any work involving: ov cmd command, running commands in containers, or container exec with notifications.
---

# ov cmd -- Single Command Execution

## Overview

Runs a single command inside a running container via `<engine> exec` and sends a D-Bus desktop notification when the command completes. Useful for one-off commands without opening an interactive shell.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Run command | `ov cmd <image> "command"` | Execute command in running container |
| With instance | `ov cmd <image> "command" -i 2` | Target specific instance |
| With env vars | `ov cmd <image> "command" -e KEY=VALUE` | Pass environment variables |

## Usage

```bash
# Run a command in a running container
ov cmd fedora "uname -a"

# Run with environment variables
ov cmd jupyter "python script.py" -e CUDA_VISIBLE_DEVICES=0

# Target a specific instance
ov cmd ollama "ollama list" -i 2
```

## Flags

| Flag | Description |
|------|-------------|
| `-i, --instance INSTANCE` | Target a specific container instance (default: 1) |
| `-e, --env KEY=VALUE` | Pass environment variables to the command |

## Behavior

- The container must already be running (`ov start <image>`)
- Agent forwarding (SSH/GPG) env vars are injected automatically if enabled
- A D-Bus desktop notification is sent on command completion (success or failure)
- For interactive shells, use `ov shell` instead
- For persistent sessions or long-running commands, use `ov tmux cmd`

## Cross-References

- `/ov-build:pull` -- Prerequisite: fetch the image into local storage; handles remote refs (`@github.com/...`) and the `ErrImageNotLocal` recovery path

- `/ov-core:shell` -- Interactive shell sessions with workspace mounts
- `/ov-advanced:dbus` -- D-Bus interaction inside containers
- `/ov-advanced:tmux` -- Persistent tmux sessions for long-running commands
