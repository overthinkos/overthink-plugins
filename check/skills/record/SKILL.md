---
name: record
description: |
  Record terminal sessions (asciinema) or desktop video (pixelflux/wf-recorder).
  MUST be invoked before any work involving: charly check record commands, terminal recording,
  desktop video recording, or session capture.
---

# Record -- Terminal and Desktop Recording

## Overview

`charly check record` manages recording sessions inside running containers. It is one
of the live-container verbs under `charly check` (see `/charly-check:check` for the full
family: cdp/wl/dbus/vnc/mcp/spice/libvirt/record). It supports two modes:

1. **Terminal recording** (asciinema) — Records terminal sessions as `.cast` files
2. **Desktop video recording** — Records full-screen video as MP4 files via pixelflux (selkies-desktop) or wf-recorder (sway-desktop)

All recording sessions are managed via tmux sessions with a `record-` prefix. This provides background execution, output monitoring, and clean start/stop lifecycle.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Start recording | `charly check record start <image> [-n NAME] [-m MODE]` | Start a recording session |
| Stop recording | `charly check record stop <image> [-n NAME] [-o FILE]` | Stop and optionally copy to host |
| List recordings | `charly check record list <image>` | Show active recording sessions |
| Send command | `charly check record cmd <image> "command" [-n NAME]` | Send command to recording terminal |

## Commands

### `charly check record start` — Start Recording

```bash
charly check record start <image> [--name NAME] [--mode MODE] [--fps FPS] [--audio] [-i INSTANCE]
```

**Flags:**
- `-n, --name` — Recording name (default: `default`). Multiple concurrent recordings supported
- `-m, --mode` — `terminal` (asciinema), `desktop` (video), or `auto` (auto-detect)
- `--fps` — Frames per second for desktop recording (default: 30)
- `--audio` — Include PulseAudio audio capture (desktop mode)

**Auto-detection order:** pixelflux-record → wf-recorder → asciinema

**Output files** (inside container):
- Terminal: `/tmp/charly-recordings/<name>.cast`
- Desktop: `/tmp/charly-recordings/<name>.mp4`

### `charly check record stop` — Stop Recording

```bash
charly check record stop <image> [--name NAME] [--output FILE] [-i INSTANCE]
```

- Sends graceful stop signal (exit for asciinema, SIGINT for video recorders)
- Waits up to 5s for graceful shutdown, then force-kills
- `-o, --output` copies the recording file from the container to the host

### `charly check record list` — List Active Recordings

```bash
charly check record list <image> [-i INSTANCE]
```

Shows all active recording sessions with name, mode, and file path.

### `charly check record cmd` — Send Command to Recording

```bash
charly check record cmd <image> "command" [--name NAME] [-i INSTANCE]
```

Sends a command to the recording's tmux session. For terminal recordings, the command and its output become part of the `.cast` file. Convenience wrapper for `charly tmux send <image> -s record-<name> "command" --enter`. Sends a desktop notification when the command is dispatched (if a notification daemon like swaync is running).

## Recording Tools

| Tool | Layer | Desktop | Protocol |
|------|-------|---------|----------|
| asciinema | `asciinema` (or `dev-tools`) | N/A | Terminal capture |
| pixelflux-record | `wl-record-pixelflux` | selkies-desktop | selkies WebSocket capture bridge → H.264 → ffmpeg |
| wf-recorder | `wf-recorder` | sway-desktop | wlr-screencopy |

## Use Case: Terminal Demo Recording

```bash
# Start recording
charly check record start openclaw -n demo --mode terminal

# Run commands (each appears in the recording)
charly check record cmd openclaw "echo 'Hello World'" -n demo
charly check record cmd openclaw "ls -la" -n demo
charly check record cmd openclaw "charly --help" -n demo

# Or attach interactively
charly tmux attach openclaw -s record-demo
# ... interact, Ctrl-b d to detach

# Stop and save
charly check record stop openclaw -n demo -o demo.cast

# Play back locally
asciinema play demo.cast
```

## Use Case: Desktop Walkthrough Video

```bash
# Start desktop video recording with audio
charly check record start selkies-desktop -n walkthrough --mode desktop --audio

# Run CLI commands (visible in recording)
charly check record cmd selkies-desktop "git status" -n walkthrough
charly check record cmd selkies-desktop "docker ps" -n walkthrough

# Interact with browser
charly check cdp open selkies-desktop "https://github.com"
charly check wl click selkies-desktop 640 360

# Stop and save
charly check record stop selkies-desktop -n walkthrough -o walkthrough.mp4
```

## Prerequisites

- **Terminal recording:** `asciinema` layer (or `dev-tools`)
- **Desktop recording (selkies):** `wl-record-pixelflux` layer (included in `selkies-desktop` metalayer)
- **Desktop recording (sway):** `wf-recorder` layer (included in `sway-desktop` metalayer)
- **All modes:** `tmux` layer must be present (for session management)

## Implementation Notes

- `charly check record cmd` uses the shared `sendTmuxCommand()` helper (also used by `charly tmux cmd`)
- Desktop notifications are sent via `sendContainerNotification()` on every `charly check record cmd` dispatch (always-on, no `--no-notify` flag — recording context always benefits from notification)
- The notification chain: `sendContainerNotification()` → tries in-container `charly check dbus notify .` → falls back to `gdbus call` → silently skips if neither available

## Cross-References

- `/charly-automation:tmux` — Underlying tmux session management (recording sessions use `record-` prefix). `charly tmux cmd` uses the same `sendTmuxCommand` helper
- `/charly-coder:asciinema` — Terminal recording layer
- `/charly-selkies:wl-record-pixelflux` — Pixelflux video recording layer
- `/charly-selkies:wf-recorder` — wf-recorder video recording layer
- `/charly-selkies:selkies-desktop-layer` — Desktop metalayer with pixelflux recording
- `/charly-selkies:sway-desktop` — Desktop metalayer with wf-recorder
- `/charly-infrastructure:dbus-layer` — D-Bus session bus (required for notification delivery)
- `/charly-selkies:swaync` — Notification daemon (displays the notifications)
- `/charly-check:wl` — Desktop automation (used alongside recording)
- `/charly-check:wl-overlay` — Fullscreen overlays (title cards, lower-thirds, fades — compose with recording workflow)
- `/charly-check:cdp` — Chrome automation (used alongside recording)

## When to Use This Skill

**MUST be invoked** when the task involves:

- Recording terminal sessions or desktop video
- `charly check record` commands
- Creating demo videos or walkthroughs
- Capturing asciinema sessions
- "How do I record my desktop?"
- "How do I make a demo video?"
