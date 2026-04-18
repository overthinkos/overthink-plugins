---
name: tmux
description: |
  Persistent tmux sessions inside containers: shell reconnection, background commands,
  output capture, and key sending. Use when running long-lived or TTY-dependent commands.
  MUST be invoked before any work involving: ov tmux commands, persistent shells,
  background container commands, or TTY-dependent TUI programs.
---

# Tmux - Persistent Sessions Inside Containers

## Overview

`ov tmux` manages tmux sessions inside running containers. It solves two problems:

1. **Persistent shells** — SSH drops, terminal closes, network interruption? Just run `ov tmux shell <image>` again and you're back where you left off.
2. **TTY-dependent background commands** — TUI programs like `openclaw models auth login` need a real terminal to function. `ov shell --tty` piped through `tee` or redirected breaks the TUI event loop. `ov tmux run` provides a real tmux terminal that the TUI can use, while `ov tmux capture` reads the output without attaching.

All operations translate to `engine exec container tmux <args>`. The tmux layer must be present in the image (`layers/tmux/layer.yml` — installs the `tmux` RPM).

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Persistent shell | `ov tmux shell <image>` | Create or reattach to a shell session |
| Send command | `ov tmux cmd <image> "command" -s <name>` | Send command to session (with notification) |
| Run detached | `ov tmux run <image> -s <name> "<command>"` | Start command in new detached session |
| Attach | `ov tmux attach <image> -s <name>` | Attach to session interactively |
| List sessions | `ov tmux list <image>` | Show active tmux sessions |
| Capture output | `ov tmux capture <image> -s <name>` | Print pane output (for automation) |
| Send keys | `ov tmux send <image> -s <name> "text" --enter` | Send keystrokes to session |
| Kill session | `ov tmux kill <image> -s <name>` | Kill a tmux session |

All commands accept `-i INSTANCE` for multi-instance support.

## Commands

### `ov tmux shell` — Persistent Shell

The user-friendly entry point. Creates or reattaches to a persistent shell session.

```bash
# First call creates a "shell" session and attaches
ov tmux shell openclaw-ollama-sway-browser

# After detaching (Ctrl-b d) or disconnect, reattach:
ov tmux shell openclaw-ollama-sway-browser
# → Picks up right where you left off

# Use a custom session name:
ov tmux shell openclaw-ollama-sway-browser -s dev
```

**Behavior:**
- If session exists → `tmux attach-session -t <name>`
- If not → `tmux new-session -s <name>` (creates with bash)
- Always interactive — uses `syscall.Exec` with `-it` for a real terminal
- Default session name: `shell`
- Detach with **Ctrl-b d** (standard tmux detach)

### `ov tmux cmd` — Send Command to Session

Sends a command (text + Enter) to an existing tmux session. Returns immediately after sending. Sends a desktop notification by default (disable with `--no-notify`).

```bash
ov tmux cmd openclaw-ollama-sway-browser "ls -la" -s oauth
ov tmux cmd jupyter "python train.py" -s training --no-notify
```

**Behavior:**
- Sends literal text followed by Enter key to the named session
- Errors if session doesn't exist (use `ov tmux list` to see sessions)
- Desktop notification sent when command is dispatched (requires swaync/mako daemon)
- Convenience wrapper for `ov tmux send <image> -s <name> -l "command" --enter`

### `ov tmux run` — Detached Command

Starts a command in a new named tmux session. Returns immediately — the command runs in the background with a real TTY.

```bash
# Run OAuth flow in background with real terminal
ov tmux run openclaw-ollama-sway-browser -s oauth \
  "openclaw models auth login --provider openai-codex --set-default"

# Run any long-lived command
ov tmux run jupyter -s training "python train.py --epochs 100"
```

**Behavior:**
- Non-interactive — uses `exec.Command`, returns immediately
- Creates session with `-d` (detached)
- Errors if session name already exists
- The command runs inside a real tmux PTY — TUI programs work correctly

### `ov tmux attach` — Interactive Attach

Attaches to an existing session. Use when you want to interact with a running command.

```bash
ov tmux attach openclaw-ollama-sway-browser -s oauth
# Detach with Ctrl-b d
```

**Behavior:**
- Interactive — `syscall.Exec` replaces the process
- Errors if session not found (use `ov tmux list` to see sessions)

### `ov tmux list` — List Sessions

```bash
ov tmux list openclaw-ollama-sway-browser
# shell: 1 windows (created Sat Mar 21 16:54:15 2026)
# oauth: 1 windows (created Sat Mar 21 17:01:22 2026)
```

No sessions is not an error — prints an informational message.

### `ov tmux capture` — Read Output

Reads the current pane content without attaching. Essential for automation — Claude Code and scripts can check on running commands.

```bash
# Read visible pane content
ov tmux capture openclaw-ollama-sway-browser -s oauth

# Read last 50 lines of history
ov tmux capture openclaw-ollama-sway-browser -s oauth -n 50
```

**Flags:**
- `-n <lines>` — Number of history lines (0 = visible pane only, default)

### `ov tmux send` — Send Keys

Sends keystrokes to a running session. For automation when you need to respond to prompts.

```bash
# Type text and press Enter
ov tmux send openclaw-ollama-sway-browser -s oauth "yes" --enter

# Send literal text (disable tmux key name interpretation)
ov tmux send openclaw-ollama-sway-browser -s oauth -l "C-c is not ctrl-c here"

# Send special keys
ov tmux send openclaw-ollama-sway-browser -s oauth Enter
ov tmux send openclaw-ollama-sway-browser -s oauth C-c
```

**Flags:**
- `--enter` — Append Enter key after the text
- `-l` / `--literal` — Send keys literally (disable key name lookup)

### `ov tmux kill` — Kill Session

```bash
ov tmux kill openclaw-ollama-sway-browser -s oauth
# Killed tmux session "oauth" in ov-openclaw-ollama-sway-browser
```

## Use Case: OpenAI Codex OAuth

The primary use case that motivated `ov tmux`. The `openclaw models auth login` TUI requires a real terminal to complete the post-callback token exchange. `ov shell --tty` piped through `tee` or backgrounded breaks the TUI event loop — the callback is received but tokens are never saved.

```bash
IMG=openclaw-ollama-sway-browser

# 1. Start OAuth in a tmux session (real terminal)
ov tmux run $IMG -s oauth \
  "openclaw models auth login --provider openai-codex --set-default"

# 2. Read the OAuth URL from tmux output
ov tmux capture $IMG -s oauth | grep -o 'https://auth.openai.com/[^ ]*'

# 3. Open URL in Chrome, complete OAuth via CDP
ov cdp open $IMG "<oauth-url>"
TAB=$(ov cdp list $IMG | grep -i "openai" | head -1 | awk '{print $1}')
ov cdp click $IMG $TAB 'button._buttonStyleFix_wvuha_65' --vnc   # Continue with Google
sleep 5
ov cdp click $IMG $TAB 'button._primary_3rdp0_107' --vnc          # Continue (consent)

# 4. Verify token exchange completed
sleep 10
ov tmux capture $IMG -s oauth | tail -5
# Should show: "OpenAI OAuth complete", "Default model set to openai-codex/gpt-5.4"

# 5. Clean up
ov tmux kill $IMG -s oauth
```

## Use Case: Persistent Development Shell

```bash
# Start a shell that survives disconnects
ov tmux shell jupyter

# Work inside the shell...
# pip install something, edit configs, run scripts

# Connection drops or terminal closes
# Just reconnect:
ov tmux shell jupyter
# → Everything is still there
```

## Prerequisites

The `tmux` layer must be included in the image. Check with:

```bash
ov shell <image> -c "which tmux"
```

If tmux is not installed, `ov tmux` returns: `"tmux is not installed in container <name> (add the tmux layer to your image)"`.

The tmux layer is already a dependency of `openclaw-full` (and all derivative images). For other images, add `tmux` to the layers list in `image.yml`.

## tmux Basics

For users unfamiliar with tmux:

| Key | Action |
|-----|--------|
| `Ctrl-b d` | Detach from session (leaves it running) |
| `Ctrl-b [` | Enter scroll mode (navigate with arrows, q to exit) |
| `Ctrl-b c` | Create new window |
| `Ctrl-b n` / `Ctrl-b p` | Next / previous window |
| `Ctrl-b %` | Split pane vertically |
| `Ctrl-b "` | Split pane horizontally |

## Naming Consistency

Command execution follows a consistent naming pattern across ov:

| Interactive shell | Single command | Persistent session |
|-------------------|---------------|-------------------|
| `ov shell` | `ov cmd` | — |
| `ov tmux shell` | `ov tmux cmd` | `ov tmux run` |

- `ov cmd` — runs command synchronously in running container, notifies on completion
- `ov tmux cmd` — sends command to existing tmux session, notifies on dispatch
- `ov tmux run` — creates new detached tmux session with command

## Cross-References

- `ov cmd` — Single command execution with D-Bus notification (running containers only, no tmux)
- `/ov:shell` — `ov shell` for one-shot commands (no persistence) or `ov shell -c` (full container setup)
- `/ov:cdp` — Chrome DevTools Protocol (used with tmux for OAuth flows)
- `/ov:openclaw` — OpenClaw gateway config (OAuth requires tmux for token exchange)
- `/ov:service` — Supervisord service management (different scope: persistent services vs ad-hoc commands)
- `/ov-layers:tmux` — The tmux layer definition
- `/ov-layers:dbus` — D-Bus session bus (required for notifications)

## When to Use This Skill

**MUST be invoked** when the task involves:

- Running long-lived commands inside containers
- TTY-dependent TUI programs (OAuth flows, interactive installers)
- Persistent shells that survive disconnects
- Background commands with output monitoring
- Sending keystrokes to running programs
- "How do I run a command that needs a real terminal?"
- "How do I reconnect to a container shell after disconnect?"
- "How do I run openclaw auth login in the background?"
