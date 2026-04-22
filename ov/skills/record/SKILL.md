---
name: record
description: |
  Record terminal sessions (asciinema) or desktop video (pixelflux/wf-recorder).
  MUST be invoked before any work involving: ov test record commands, terminal recording,
  desktop video recording, or session capture.
---

# Record -- Terminal and Desktop Recording

## Overview

`ov test record` manages recording sessions inside running containers. It supports two modes:

> **Moved under `ov test` in 2026-04** — the top-level `ov record` command was removed as a hard cutover. See `/ov:test` for the live-container verb family (cdp/wl/dbus/vnc/mcp/spice/libvirt/record).

1. **Terminal recording** (asciinema) — Records terminal sessions as `.cast` files
2. **Desktop video recording** — Records full-screen video as MP4 files via pixelflux (selkies-desktop) or wf-recorder (sway-desktop)

All recording sessions are managed via tmux sessions with a `record-` prefix. This provides background execution, output monitoring, and clean start/stop lifecycle.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Start recording | `ov test record start <image> [-n NAME] [-m MODE]` | Start a recording session |
| Stop recording | `ov test record stop <image> [-n NAME] [-o FILE]` | Stop and optionally copy to host |
| List recordings | `ov test record list <image>` | Show active recording sessions |
| Send command | `ov test record cmd <image> "command" [-n NAME]` | Send command to recording terminal |

## Commands

### `ov test record start` — Start Recording

```bash
ov test record start <image> [--name NAME] [--mode MODE] [--fps FPS] [--audio] [-i INSTANCE]
```

**Flags:**
- `-n, --name` — Recording name (default: `default`). Multiple concurrent recordings supported
- `-m, --mode` — `terminal` (asciinema), `desktop` (video), or `auto` (auto-detect)
- `--fps` — Frames per second for desktop recording (default: 30)
- `--audio` — Include PulseAudio audio capture (desktop mode)

**Auto-detection order:** pixelflux-record → wf-recorder → asciinema

**Output files** (inside container):
- Terminal: `/tmp/ov-recordings/<name>.cast`
- Desktop: `/tmp/ov-recordings/<name>.mp4`

### `ov test record stop` — Stop Recording

```bash
ov test record stop <image> [--name NAME] [--output FILE] [-i INSTANCE]
```

- Sends graceful stop signal (exit for asciinema, SIGINT for video recorders)
- Waits up to 5s for graceful shutdown, then force-kills
- `-o, --output` copies the recording file from the container to the host

### `ov test record list` — List Active Recordings

```bash
ov test record list <image> [-i INSTANCE]
```

Shows all active recording sessions with name, mode, and file path.

### `ov test record cmd` — Send Command to Recording

```bash
ov test record cmd <image> "command" [--name NAME] [-i INSTANCE]
```

Sends a command to the recording's tmux session. For terminal recordings, the command and its output become part of the `.cast` file. Convenience wrapper for `ov tmux send <image> -s record-<name> "command" --enter`. Sends a desktop notification when the command is dispatched (if a notification daemon like swaync is running).

## Recording Tools

| Tool | Layer | Desktop | Protocol |
|------|-------|---------|----------|
| asciinema | `asciinema` (or `dev-tools`) | N/A | Terminal capture |
| pixelflux-record | `wl-record-pixelflux` | selkies-desktop | selkies WebSocket capture bridge → H.264 → ffmpeg |
| wf-recorder | `wf-recorder` | sway-desktop | wlr-screencopy |

## Use Case: Terminal Demo Recording

```bash
# Start recording
ov test record start openclaw -n demo --mode terminal

# Run commands (each appears in the recording)
ov test record cmd openclaw "echo 'Hello World'" -n demo
ov test record cmd openclaw "ls -la" -n demo
ov test record cmd openclaw "ov --help" -n demo

# Or attach interactively
ov tmux attach openclaw -s record-demo
# ... interact, Ctrl-b d to detach

# Stop and save
ov test record stop openclaw -n demo -o demo.cast

# Play back locally
asciinema play demo.cast
```

## Use Case: Desktop Walkthrough Video

```bash
# Start desktop video recording with audio
ov test record start selkies-desktop -n walkthrough --mode desktop --audio

# Run CLI commands (visible in recording)
ov test record cmd selkies-desktop "git status" -n walkthrough
ov test record cmd selkies-desktop "docker ps" -n walkthrough

# Interact with browser
ov test cdp open selkies-desktop "https://github.com"
ov test wl click selkies-desktop 640 360

# Stop and save
ov test record stop selkies-desktop -n walkthrough -o walkthrough.mp4
```

## Prerequisites

- **Terminal recording:** `asciinema` layer (or `dev-tools`)
- **Desktop recording (selkies):** `wl-record-pixelflux` layer (included in `selkies-desktop` metalayer)
- **Desktop recording (sway):** `wf-recorder` layer (included in `sway-desktop` metalayer)
- **All modes:** `tmux` layer must be present (for session management)

## Implementation Notes

- `ov test record cmd` uses the shared `sendTmuxCommand()` helper (also used by `ov tmux cmd`)
- Desktop notifications are sent via `sendContainerNotification()` on every `ov test record cmd` dispatch (always-on, no `--no-notify` flag — recording context always benefits from notification)
- The notification chain: `sendContainerNotification()` → tries in-container `ov test dbus notify .` → falls back to `gdbus call` → silently skips if neither available

## Cross-References

- `/ov:tmux` — Underlying tmux session management (recording sessions use `record-` prefix). `ov tmux cmd` uses the same `sendTmuxCommand` helper
- `/ov-layers:asciinema` — Terminal recording layer
- `/ov-layers:wl-record-pixelflux` — Pixelflux video recording layer
- `/ov-layers:wf-recorder` — wf-recorder video recording layer
- `/ov-layers:selkies-desktop` — Desktop metalayer with pixelflux recording
- `/ov-layers:sway-desktop` — Desktop metalayer with wf-recorder
- `/ov-layers:dbus` — D-Bus session bus (required for notification delivery)
- `/ov-layers:swaync` — Notification daemon (displays the notifications)
- `/ov:wl` — Desktop automation (used alongside recording)
- `/ov:wl-overlay` — Fullscreen overlays (title cards, lower-thirds, fades — compose with recording workflow)
- `/ov:cdp` — Chrome automation (used alongside recording)

## When to Use This Skill

**MUST be invoked** when the task involves:

- Recording terminal sessions or desktop video
- `ov test record` commands
- Creating demo videos or walkthroughs
- Capturing asciinema sessions
- "How do I record my desktop?"
- "How do I make a demo video?"
