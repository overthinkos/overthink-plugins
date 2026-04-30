---
name: vnc
description: |
  MUST be invoked before any work involving: VNC automation, ov eval vnc commands, RFB protocol desktop interaction, VNC screenshots, clicking coordinates, or VNC authentication.
---

# VNC - VNC Desktop Automation

## Overview

`ov eval vnc` commands connect to VNC servers (RFB protocol on port tcp:5900) inside running containers. Provides screenshot capture, keyboard/mouse input, and VNC password management for Wayland desktop automation via wayvnc.

### Also as a declarative verb

Every `ov eval vnc <method>` (status/screenshot/click/mouse/type/key/rfb/passwd) is authorable as a `vnc:` verb inside a `tests:` block. Method-specific fields (`x`, `y`, `text`, `key`, `artifact`, `artifact_min_bytes`) are siblings of the verb line. See `/ov-build:eval` for the full YAML shape. Example: `- vnc: screenshot\n  artifact: /tmp/vnc.png\n  artifact_min_bytes: 5000`.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Screenshot | `ov eval vnc screenshot <image> [file]` | Capture VNC framebuffer as PNG |
| Click | `ov eval vnc click <image> <x> <y>` | Click at x,y coordinates |
| Type text | `ov eval vnc type <image> <text>` | Send keyboard input as key events |
| Send key | `ov eval vnc key <image> <key-name>` | Press a special key (Return, Escape, etc.) |
| Move mouse | `ov eval vnc mouse <image> <x> <y>` | Move mouse without clicking |
| Status | `ov eval vnc status <image>` | Check VNC server, show resolution and desktop name |
| Set password | `ov eval vnc passwd <image>` | Set VNC auth password for deployment |
| Raw RFB | `ov eval vnc rfb <image> <method> [json]` | Send raw RFB protocol message |

## Architecture

```
CLI command -> resolveVNCContainer (engine + container name)
           -> resolveVNCAddress (docker/podman port <name> 5900)
           -> resolveVNCPassword (ov settings + VNC_PASSWORD env)
           -> NewVNCClient(address, password) -> RFB handshake -> operation
```

Custom RFC 6143 VNC client implementation (no external dependency). Supports None, VNC auth (DES), and VeNCrypt (TLS + sub-auth) security types.

## Requirements

- Container must include the `wayvnc` layer (port tcp:5900)
- Container must be running (`ov start`)
- Wayland compositor must be active (sway)

## Commands

### Screenshot
```bash
ov eval vnc screenshot openclaw-sway-browser              # saves screenshot.png
ov eval vnc screenshot openclaw-sway-browser desktop.png   # custom filename
ov eval vnc screenshot openclaw-sway-browser -i prod       # specific instance
```

### Click
```bash
ov eval vnc click openclaw-sway-browser 960 540             # left click at center of 1920x1080
ov eval vnc click openclaw-sway-browser 100 200 --button right  # right click
ov eval vnc click openclaw-sway-browser 100 200 --button middle # middle click
ov eval vnc click openclaw-sway-browser 100 200 --from-cdp $TAB   # translate from CDP viewport
ov eval vnc click openclaw-sway-browser 100 200 --from-sway google-chrome  # translate from sway window
ov eval vnc click openclaw-sway-browser 100 200 --from-x11 Steam  # translate from X11 window (XWayland)
```

**`--from-x11 <class-or-title>`** translates coordinates from X11 window-internal space to desktop-absolute VNC coordinates. Works the same as `ov eval wl click --from-x11` -- queries X11 geometry via xdotool, finds the sway node, and scales to desktop coordinates. Essential for XWayland windows (Steam, Heroic) where the X11 resolution differs from the compositor resolution.

### Type
```bash
ov eval vnc type openclaw-sway-browser "hello world"    # types each character as key events
```
Only supports ASCII/Latin-1 characters. For special keys, use `ov eval vnc key`.

### Key
```bash
ov eval vnc key openclaw-sway-browser Return       # press Enter
ov eval vnc key openclaw-sway-browser Escape       # press Escape
ov eval vnc key openclaw-sway-browser Tab          # press Tab
ov eval vnc key openclaw-sway-browser F5           # press F5
ov eval vnc key openclaw-sway-browser Control_L    # press left Ctrl
```

Valid key names: Return, Escape, Tab, BackSpace, Delete, Home, End, Page_Up, Page_Down, Up, Down, Left, Right, Insert, F1-F12, Shift_L, Shift_R, Control_L, Control_R, Alt_L, Alt_R, Super_L, Super_R, Meta_L, Meta_R, Caps_Lock, space.

### Mouse
```bash
ov eval vnc mouse openclaw-sway-browser 500 300    # move mouse to (500, 300)
```

### Status
```bash
ov eval vnc status openclaw-sway-browser
# Output:
# Desktop:    sway
# Resolution: 1920x1080
```

### Password
```bash
ov eval vnc passwd openclaw-sway-browser              # prompts for password
ov eval vnc passwd openclaw-sway-browser --generate   # generates random password, prints to stdout
```

Sets up VNC authentication (VeNCrypt/TLS):
1. Stores password in system keyring or config file (depending on `secret_backend` setting) as `vnc.password.<image>`
2. Resolves `$HOME` inside container for absolute config paths
3. Generates self-signed TLS cert+key (valid 3650 days) if not present
4. Generates RSA key in traditional format (`-traditional` flag for OpenSSL 3.x) if not present
5. Writes `~/.config/wayvnc/config` with `enable_auth=true` (wayvnc reads this automatically)
6. Restarts wayvnc supervisord service

After setting a password, all `ov eval vnc` commands authenticate transparently via VeNCrypt/TLS.

### Password Resolution Chain

When connecting, password is resolved in this order:
1. `VNC_PASSWORD` environment variable (CI/automation override)
2. System keyring lookup for `vnc.password.<image>-<instance>` (when `secret_backend=auto` or `keyring`)
3. Config file lookup for `vnc.password.<image>-<instance>` (instance-specific)
4. System keyring / config file lookup for `vnc.password.<image>` (image-level)
5. Empty string (no auth — server must allow unauthenticated connections)

```bash
# One-off password override via env
VNC_PASSWORD=secret ov eval vnc screenshot openclaw-sway-browser out.png

# Set password programmatically (alternative to ov eval vnc passwd)
ov settings set vnc.password.openclaw-sway-browser mysecret

# Instance-specific password
ov settings set vnc.password.openclaw-sway-browser-prod prodpassword
```

Requires `openssl` inside the container for TLS cert and RSA key generation.

### Raw RFB
```bash
ov eval vnc rfb openclaw-sway-browser key '{"key": 65293, "down": true}'           # raw key event
ov eval vnc rfb openclaw-sway-browser pointer '{"x": 100, "y": 200, "button": 1}'  # raw pointer
ov eval vnc rfb openclaw-sway-browser cut-text '{"text": "clipboard"}'              # clipboard
ov eval vnc rfb openclaw-sway-browser fbupdate-request                              # get dimensions
```

## Differences from CDP Commands

| Aspect | `ov eval cdp` (CDP) | `ov eval vnc` (RFB) |
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

Some websites (notably Google sign-in) detect and block CDP-based input. VNC provides a reliable fallback because `ov eval vnc type` sends real X11 keysym events through the Wayland compositor — indistinguishable from physical keyboard input.

**CDP + VNC Hybrid Pattern:** Use `ov eval cdp click --vnc` for clicking (CDP selector precision + VNC pointer delivery) and `ov eval vnc type` for typing credentials:

```bash
# --vnc click: CDP finds element by selector, delivers click via VNC pointer
ov eval cdp click my-app $TAB '#identifierId' --vnc
sleep 0.5                                          # let compositor process focus
# VNC type sends real key events through the compositor
ov eval vnc type my-app "$GMAIL_USER"
```

**Tested timing:** 500ms sleep between `--vnc` click and VNC type is sufficient. No characters were dropped at this timing during Google sign-in testing.

When to use `--vnc` click and VNC type:
- **`chrome://` pages** (required): CDP mouse events and JS `.click()` are blocked on Chrome's privileged pages (`chrome://intro/`, `chrome://sync-confirmation/`, `chrome://settings/`). `--vnc` is the only way to click.
- Google sign-in or other anti-automation-protected forms
- Sites that validate input event sequences (keyDown/keyPress/input/keyUp)
- Any form where CDP type fails silently (value appears but form doesn't accept it)

**Chrome first-run dialogs:** On fresh profiles, Chrome opens a first-run dialog as a separate window invisible to CDP. Dismiss with `ov eval wl sway msg my-app 'focus left'` then `ov eval vnc key my-app Return`.

See `/ov-advanced:cdp` for the full Google sign-in recipe.

## Using CDP Coordinates with VNC

VNC uses desktop-absolute coordinates, while CDP returns viewport-relative coordinates. Use the `--from-cdp` or `--from-sway` flags to explicitly translate:

**`--from-cdp <tab-id>`** — Translates viewport coords to desktop coords via CDP's `window.screenX/screenY`:

```bash
# Get viewport coords from ov eval cdp coords, then click via VNC
ov eval vnc click my-app 1220 328 --from-cdp $TAB
# Translated viewport (1220, 328) → desktop (1220, 439) via CDP tab ...
```

**`--from-sway <app-id>`** — Translates window-relative coords to desktop coords via sway tree:

```bash
ov eval vnc click my-app 500 200 --from-sway google-chrome
# Translated window-relative (500, 200) → desktop (504, 204) via sway app_id=google-chrome
```

Without flags, X and Y are desktop-absolute coordinates (the default, unchanged behavior).

## NVIDIA Headless: VNC Screenshots Work

VNC screenshots work correctly on NVIDIA headless for images using `sway-desktop-vnc` (the standard VNC composition). Two fixes enable this:

1. **Pixman renderer** — `sway-desktop-vnc` forces `WLR_RENDERER=pixman` (software rendering), producing buffers wayvnc can reliably capture
2. **DPMS workaround** — `wayvnc-wrapper` triggers the missing headless power event that wayvnc 0.9.1 waits for before starting capture

Both `ov eval vnc screenshot` and `ov eval wl screenshot` work on NVIDIA headless:
```bash
ov eval vnc screenshot <image> out.png           # VNC screenshot (works with pixman + DPMS fix)
ov eval wl screenshot <image> out.png            # Wayland screenshot (grim, always works)
```

## Cross-References

- `/ov-build:eval` — parent router; `ov eval vnc …` is how every invocation is dispatched.
- `/ov-advanced:wl` — Wayland-native desktop automation (sibling verb; works on NVIDIA headless).
- `/ov-advanced:cdp` — Chrome DevTools Protocol automation (sibling verb; same container, different protocol).
- `/ov-advanced:dbus` — D-Bus calls and desktop notifications (sibling verb under `ov test`).
- `/ov-advanced:wl` (sway subgroup) — Sway compositor control (window management, workspaces)
- `/ov-core:config` — VNC password storage, `secret_backend` setting, `migrate-secrets` command
- `/ov-core:service` — Managing wayvnc supervisord service
- `/ov-core:deploy` — VNC password setup in deployment workflows
- `/ov-core:shell` — Executing commands inside containers
- `/ov-build:layer` — wayvnc layer configuration (port tcp:5900)

## When to Use This Skill

**MUST be invoked** when the task involves VNC automation, ov eval vnc commands, RFB protocol desktop interaction, VNC screenshots, clicking coordinates, or VNC authentication. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Desktop automation. Use for pixel-level interaction when CDP can't reach the element. See also `/ov-advanced:cdp` (DOM, preferred), `/ov-advanced:wl` (sway subgroup) (window).
