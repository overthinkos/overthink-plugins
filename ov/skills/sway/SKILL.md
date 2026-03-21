---
name: sway
description: |
  MUST be invoked before any work involving: Sway compositor control, ov sway commands, window management, workspaces, outputs, or application launching inside containers.
---

# Sway - Compositor Control

## Overview

`ov sway` commands interact with the Sway compositor inside running containers via `swaymsg`. Provides window management (focus, move, resize, kill, float, fullscreen), layout control, workspace switching, output configuration, and application launching. All commands auto-discover the `SWAYSOCK` path inside the container.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Run command | `ov sway msg <image> <command>` | Generic swaymsg passthrough |
| Window tree | `ov sway tree <image>` | Get window/container tree (JSON) |
| Workspaces | `ov sway workspaces <image>` | List workspaces (JSON) |
| Outputs | `ov sway outputs <image>` | List outputs (JSON) |
| Launch app | `ov sway exec <image> <command>` | Execute an application |
| Focus | `ov sway focus <image> <target>` | Focus by direction or criteria |
| Move | `ov sway move <image> <target...>` | Move window (direction/scratchpad/workspace) |
| Resize | `ov sway resize <image> <dim> <amount>` | Resize window |
| Close | `ov sway kill <image>` | Close focused window |
| Fullscreen | `ov sway fullscreen <image>` | Toggle fullscreen |
| Float | `ov sway floating <image>` | Toggle floating |
| Layout | `ov sway layout <image> <mode>` | Set layout mode |
| Workspace | `ov sway workspace <image> <num>` | Switch workspace |
| Reload | `ov sway reload <image>` | Reload sway config |
| Resolution | `ov sway resolution <image> <WxH>` | Set output resolution |

All commands accept `-i INSTANCE` for multi-instance support. Use `.` as image name for local/in-container execution.

## Requirements

- Container must include the `sway` layer
- Container must be running (`ov start` or `ov enable`)
- Sway compositor must be active (supervisord service)

## Commands

### Generic Command (swaymsg passthrough)

```bash
ov sway msg my-app 'focus left'
ov sway msg my-app '[app_id=google-chrome-stable] focus'
ov sway msg my-app 'splith'
```

Passes any sway command directly to `swaymsg`. Use for commands not covered by specific subcommands.

### Query Commands

```bash
ov sway tree my-app          # Full window/container tree (JSON)
ov sway workspaces my-app    # Workspace list (JSON)
ov sway outputs my-app       # Output list (JSON)
```

Output is JSON to stdout, suitable for piping to `jq`.

### Window Management

```bash
ov sway focus my-app left                              # Focus by direction
ov sway focus my-app 'app_id=google-chrome-stable'     # Focus by criteria
ov sway move my-app right                              # Move in direction
ov sway move my-app scratchpad                         # Minimize to scratchpad
ov sway move my-app workspace 2                        # Move to workspace 2
ov sway resize my-app width 10px                       # Grow width by 10px
ov sway resize my-app width -10px                      # Shrink width by 10px
ov sway kill my-app                                    # Close focused window
ov sway fullscreen my-app                              # Toggle fullscreen
ov sway floating my-app                                # Toggle floating
ov sway layout my-app tabbed                           # Set tabbed layout
ov sway layout my-app splitv                           # Set vertical split
```

### Application Launching

```bash
ov sway exec my-app thunar                # Open file manager
ov sway exec my-app xfce4-terminal        # Open terminal
ov sway exec my-app google-chrome-stable   # Open Chrome
```

### Workspace and Display

```bash
ov sway workspace my-app 2                             # Switch to workspace 2
ov sway resolution my-app 2560x1440                    # Set resolution
ov sway resolution my-app 1920x1080 -o HEADLESS-1      # Explicit output name
ov sway reload my-app                                  # Reload config
```

## Local Mode (In-Container)

When running inside a container, use `.` as the image name:

```bash
ov sway tree .
ov sway focus . left
ov sway exec . thunar
```

This runs `swaymsg` directly instead of going through `podman exec`.

## Architecture

1. Resolves the container name from image + instance
2. Discovers `SWAYSOCK` via `ls /tmp/sway-ipc.*.sock`
3. Runs `swaymsg` with the discovered socket (via `podman exec` from host, or directly in local mode)
4. JSON output piped to stdout, errors to stderr

Source: `ov/sway.go`.

## Window Geometry for Coordinate Translation

`ov sway tree` returns window positions usable for VNC coordinate translation. Each window's `rect` field shows its desktop position and size. The `app_id` field identifies the application (e.g., `google-chrome` for Chrome).

Use `ov vnc click --from-sway <app-id>` to translate window-relative coordinates to desktop-absolute coordinates by querying the sway tree:

```bash
ov vnc click my-app 500 200 --from-sway google-chrome
# Translated window-relative (500, 200) → desktop (504, 204) via sway app_id=google-chrome
```

This is useful when you know where an element is relative to the window but need desktop coordinates for VNC.

## Cross-References

- `/ov:cdp` -- Chrome DevTools Protocol (tab management, JavaScript eval)
- `/ov:vnc` -- VNC desktop automation (pixel-level interaction)
- `/ov:layer` -- `sway` layer configuration
- `/ov:shell` -- Running commands in containers

## When to Use This Skill

**MUST be invoked** when the task involves Sway compositor control, ov sway commands, window management, workspaces, outputs, or application launching inside containers. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Desktop automation. Use for window management in running desktop containers. See also `/ov:cdp` (DOM), `/ov:vnc` (pixel).
