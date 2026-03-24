---
name: vnc
description: |
  MUST be invoked before any work involving: VNC automation, ov vnc commands, RFB protocol desktop interaction, VNC screenshots, clicking coordinates, or VNC authentication.
---

# VNC - VNC Desktop Automation

## Overview

`ov vnc` commands connect to VNC servers (RFB protocol on port tcp:5900) inside running containers. Provides screenshot capture, keyboard/mouse input, and VNC password management for Wayland desktop automation via wayvnc.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Screenshot | `ov vnc screenshot <image> [file]` | Capture VNC framebuffer as PNG |
| Click | `ov vnc click <image> <x> <y>` | Click at x,y coordinates |
| Type text | `ov vnc type <image> <text>` | Send keyboard input as key events |
| Send key | `ov vnc key <image> <key-name>` | Press a special key (Return, Escape, etc.) |
| Move mouse | `ov vnc mouse <image> <x> <y>` | Move mouse without clicking |
| Status | `ov vnc status <image>` | Check VNC server, show resolution and desktop name |
| Set password | `ov vnc passwd <image>` | Set VNC auth password for deployment |
| Raw RFB | `ov vnc rfb <image> <method> [json]` | Send raw RFB protocol message |

## Architecture

```
CLI command -> resolveVNCContainer (engine + container name)
           -> resolveVNCAddress (docker/podman port <name> 5900)
           -> resolveVNCPassword (ov config + VNC_PASSWORD env)
           -> NewVNCClient(address, password) -> RFB handshake -> operation
```

Custom RFC 6143 VNC client implementation (no external dependency). Supports None, VNC auth (DES), and VeNCrypt (TLS + sub-auth) security types.

## Requirements

- Container must include the `wayvnc` layer (port tcp:5900)
- Container must be running (`ov start` or `ov enable`)
- Wayland compositor must be active (sway)

## Commands

### Screenshot
```bash
ov vnc screenshot openclaw-sway-browser              # saves screenshot.png
ov vnc screenshot openclaw-sway-browser desktop.png   # custom filename
ov vnc screenshot openclaw-sway-browser -i prod       # specific instance
```

### Click
```bash
ov vnc click openclaw-sway-browser 960 540             # left click at center of 1920x1080
ov vnc click openclaw-sway-browser 100 200 --button right  # right click
ov vnc click openclaw-sway-browser 100 200 --button middle # middle click
ov vnc click openclaw-sway-browser 100 200 --from-cdp $TAB   # translate from CDP viewport
ov vnc click openclaw-sway-browser 100 200 --from-sway google-chrome  # translate from sway window
ov vnc click openclaw-sway-browser 100 200 --from-x11 Moonlight  # translate from X11 window (XWayland)
```

**`--from-x11 <class-or-title>`** translates coordinates from X11 window-internal space to desktop-absolute VNC coordinates. Works the same as `ov wl click --from-x11` -- queries X11 geometry via xdotool, finds the sway node, and scales to desktop coordinates. Essential for XWayland windows (Moonlight, Steam) where the X11 resolution differs from the compositor resolution.

### Type
```bash
ov vnc type openclaw-sway-browser "hello world"    # types each character as key events
```
Only supports ASCII/Latin-1 characters. For special keys, use `ov vnc key`.

### Key
```bash
ov vnc key openclaw-sway-browser Return       # press Enter
ov vnc key openclaw-sway-browser Escape       # press Escape
ov vnc key openclaw-sway-browser Tab          # press Tab
ov vnc key openclaw-sway-browser F5           # press F5
ov vnc key openclaw-sway-browser Control_L    # press left Ctrl
```

Valid key names: Return, Escape, Tab, BackSpace, Delete, Home, End, Page_Up, Page_Down, Up, Down, Left, Right, Insert, F1-F12, Shift_L, Shift_R, Control_L, Control_R, Alt_L, Alt_R, Super_L, Super_R, Meta_L, Meta_R, Caps_Lock, space.

### Mouse
```bash
ov vnc mouse openclaw-sway-browser 500 300    # move mouse to (500, 300)
```

### Status
```bash
ov vnc status openclaw-sway-browser
# Output:
# Desktop:    sway
# Resolution: 1920x1080
```

### Password
```bash
ov vnc passwd openclaw-sway-browser              # prompts for password
ov vnc passwd openclaw-sway-browser --generate   # generates random password, prints to stdout
```

Sets up VNC authentication (VeNCrypt/TLS):
1. Stores password in system keyring or config file (depending on `secret_backend` setting) as `vnc.password.<image>`
2. Resolves `$HOME` inside container for absolute config paths
3. Generates self-signed TLS cert+key (valid 3650 days) if not present
4. Generates RSA key in traditional format (`-traditional` flag for OpenSSL 3.x) if not present
5. Writes `~/.config/wayvnc/config` with `enable_auth=true` (wayvnc reads this automatically)
6. Restarts wayvnc supervisord service

After setting a password, all `ov vnc` commands authenticate transparently via VeNCrypt/TLS.

### Password Resolution Chain

When connecting, password is resolved in this order:
1. `VNC_PASSWORD` environment variable (CI/automation override)
2. System keyring lookup for `vnc.password.<image>-<instance>` (when `secret_backend=auto` or `keyring`)
3. Config file lookup for `vnc.password.<image>-<instance>` (instance-specific)
4. System keyring / config file lookup for `vnc.password.<image>` (image-level)
5. Empty string (no auth — server must allow unauthenticated connections)

```bash
# One-off password override via env
VNC_PASSWORD=secret ov vnc screenshot openclaw-sway-browser out.png

# Set password programmatically (alternative to ov vnc passwd)
ov config set vnc.password.openclaw-sway-browser mysecret

# Instance-specific password
ov config set vnc.password.openclaw-sway-browser-prod prodpassword
```

Requires `openssl` inside the container for TLS cert and RSA key generation.

### Raw RFB
```bash
ov vnc rfb openclaw-sway-browser key '{"key": 65293, "down": true}'           # raw key event
ov vnc rfb openclaw-sway-browser pointer '{"x": 100, "y": 200, "button": 1}'  # raw pointer
ov vnc rfb openclaw-sway-browser cut-text '{"text": "clipboard"}'              # clipboard
ov vnc rfb openclaw-sway-browser fbupdate-request                              # get dimensions
```

## Differences from CDP Commands

| Aspect | `ov cdp` (CDP) | `ov vnc` (RFB) |
|--------|----------------|----------------|
| Protocol | WebSocket JSON | Binary TCP |
| Scope | Browser tabs | Whole desktop |
| Click | CSS selector (viewport-relative) | x,y coordinates (desktop-absolute) |
| Type | CDP key events | Key events (keysyms) |
| Screenshot | Browser page only | Full desktop |
| JavaScript | Yes (eval/wait) | No |
| Use case | Web automation | Desktop automation |

Source: `ov/vnc_client.go`, `ov/vnc.go`.

## VNC as Anti-Detection Fallback

Some websites (notably Google sign-in) detect and block CDP-based input. VNC provides a reliable fallback because `ov vnc type` sends real X11 keysym events through the Wayland compositor — indistinguishable from physical keyboard input.

**CDP + VNC Hybrid Pattern:** Use `ov cdp click --vnc` for clicking (CDP selector precision + VNC pointer delivery) and `ov vnc type` for typing credentials:

```bash
# --vnc click: CDP finds element by selector, delivers click via VNC pointer
ov cdp click my-app $TAB '#identifierId' --vnc
sleep 0.5                                          # let compositor process focus
# VNC type sends real key events through the compositor
ov vnc type my-app "$GMAIL_USER"
```

**Tested timing:** 500ms sleep between `--vnc` click and VNC type is sufficient. No characters were dropped at this timing during Google sign-in testing.

When to use `--vnc` click and VNC type:
- **`chrome://` pages** (required): CDP mouse events and JS `.click()` are blocked on Chrome's privileged pages (`chrome://intro/`, `chrome://sync-confirmation/`, `chrome://settings/`). `--vnc` is the only way to click.
- Google sign-in or other anti-automation-protected forms
- Sites that validate input event sequences (keyDown/keyPress/input/keyUp)
- Any form where CDP type fails silently (value appears but form doesn't accept it)

**Chrome first-run dialogs:** On fresh profiles, Chrome opens a first-run dialog as a separate window invisible to CDP. Dismiss with `ov sway msg my-app 'focus left'` then `ov vnc key my-app Return`.

See `/ov:cdp` for the full Google sign-in recipe.

## Using CDP Coordinates with VNC

VNC uses desktop-absolute coordinates, while CDP returns viewport-relative coordinates. Use the `--from-cdp` or `--from-sway` flags to explicitly translate:

**`--from-cdp <tab-id>`** — Translates viewport coords to desktop coords via CDP's `window.screenX/screenY`:

```bash
# Get viewport coords from ov cdp coords, then click via VNC
ov vnc click my-app 1220 328 --from-cdp $TAB
# Translated viewport (1220, 328) → desktop (1220, 439) via CDP tab ...
```

**`--from-sway <app-id>`** — Translates window-relative coords to desktop coords via sway tree:

```bash
ov vnc click my-app 500 200 --from-sway google-chrome
# Translated window-relative (500, 200) → desktop (504, 204) via sway app_id=google-chrome
```

Without flags, X and Y are desktop-absolute coordinates (the default, unchanged behavior).

## NVIDIA Headless: VNC Gray Screen

On NVIDIA headless systems, `ov vnc screenshot` produces gray images. This is an **upstream wayvnc/neatvnc bug** — `ext-image-copy-capture` doesn't deliver frame data with gles2 on sway 1.11 / wlroots 0.19. VNC remote viewing (connecting a VNC client) may still work, but screenshots are unreliable.

**Use `ov wl` instead of `ov vnc` for screenshots on NVIDIA:**
```bash
ov wl screenshot <image>                    # Wayland screenshot (grim, works on NVIDIA)
ov wl windows <image>                       # List X11 windows
ov wl focus <image> "Moonlight"             # Focus specific X11 window
```

## Cross-references

- `/ov:wl` — Wayland-native desktop automation (works on NVIDIA headless)
- `/ov:cdp` — Chrome DevTools Protocol automation (same container, different protocol)
- `/ov:sway` — Sway compositor control (window management, workspaces)
- `/ov:config` — VNC password storage, `secret_backend` setting, `migrate-secrets` command
- `/ov:service` — Managing wayvnc supervisord service
- `/ov:deploy` — VNC password setup in deployment workflows
- `/ov:shell` — Executing commands inside containers
- `/ov:layer` — wayvnc layer configuration (port tcp:5900)

## When to Use This Skill

**MUST be invoked** when the task involves VNC automation, ov vnc commands, RFB protocol desktop interaction, VNC screenshots, clicking coordinates, or VNC authentication. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Desktop automation. Use for pixel-level interaction when CDP can't reach the element. See also `/ov:cdp` (DOM, preferred), `/ov:sway` (window).
