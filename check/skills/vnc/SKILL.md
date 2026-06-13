---
name: vnc
description: |
  MUST be invoked before any work involving: VNC automation, charly check vnc commands, RFB protocol desktop interaction, VNC screenshots, clicking coordinates, or VNC authentication.
---

# VNC - VNC Desktop Automation

## Overview

`charly check vnc` commands connect to VNC servers (RFB protocol on port tcp:5900) inside running containers. Provides screenshot capture, keyboard/mouse input, and VNC password management for Wayland desktop automation via wayvnc.

### Also as a declarative verb

Every `charly check vnc <method>` (status/screenshot/click/mouse/type/key/rfb/passwd) is authorable as a `vnc:` verb inside a `check:` block. Method-specific fields (`x`, `y`, `text`, `key`, `artifact`, `artifact_min_bytes`) are siblings of the verb line. See `/charly-check:check` for the full YAML shape. Example: `- vnc: screenshot\n  artifact: /tmp/vnc.png\n  artifact_min_bytes: 5000`.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Screenshot | `charly check vnc screenshot <image> [file]` | Capture VNC framebuffer as PNG |
| Click | `charly check vnc click <image> <x> <y>` | Click at x,y coordinates |
| Type text | `charly check vnc type <image> <text>` | Send keyboard input as key events |
| Send key | `charly check vnc key <image> <key-name>` | Press a special key (Return, Escape, etc.) |
| Move mouse | `charly check vnc mouse <image> <x> <y>` | Move mouse without clicking |
| Status | `charly check vnc status <image>` | Check VNC server, show resolution and desktop name |
| Set password | `charly check vnc passwd <image>` | Set VNC auth password for deployment |
| Raw RFB | `charly check vnc rfb <image> <method> [json]` | Send raw RFB protocol message |

## Architecture

```
CLI command -> resolveVNCContainer (engine + container name)
           -> resolveVNCAddress (docker/podman port <name> 5900)
           -> resolveVNCPassword (charly settings + VNC_PASSWORD env)
           -> NewVNCClient(address, password) -> RFB handshake -> operation
```

Custom RFC 6143 VNC client implementation (no external dependency). Supports None, VNC auth (DES), and VeNCrypt (TLS + sub-auth) security types.

## Requirements

- Container must include the `wayvnc` layer (port tcp:5900)
- Container must be running (`charly start`)
- Wayland compositor must be active (sway)

## Commands

### Screenshot
```bash
charly check vnc screenshot sway-browser-vnc              # saves screenshot.png
charly check vnc screenshot sway-browser-vnc desktop.png   # custom filename
charly check vnc screenshot sway-browser-vnc -i prod       # specific instance
```

### Click
```bash
charly check vnc click sway-browser-vnc 960 540             # left click at center of 1920x1080
charly check vnc click sway-browser-vnc 100 200 --button right  # right click
charly check vnc click sway-browser-vnc 100 200 --button middle # middle click
charly check vnc click sway-browser-vnc 100 200 --from-cdp $TAB   # translate from CDP viewport
charly check vnc click sway-browser-vnc 100 200 --from-sway google-chrome  # translate from sway window
charly check vnc click sway-browser-vnc 100 200 --from-x11 Steam  # translate from X11 window (XWayland)
```

**`--from-x11 <class-or-title>`** translates coordinates from X11 window-internal space to desktop-absolute VNC coordinates. Works the same as `charly check wl click --from-x11` -- queries X11 geometry via xdotool, finds the sway node, and scales to desktop coordinates. Essential for XWayland windows (Steam, Heroic) where the X11 resolution differs from the compositor resolution.

### Type
```bash
charly check vnc type sway-browser-vnc "hello world"    # types each character as key events
```
Only supports ASCII/Latin-1 characters. For special keys, use `charly check vnc key`.

### Key
```bash
charly check vnc key sway-browser-vnc Return       # press Enter
charly check vnc key sway-browser-vnc Escape       # press Escape
charly check vnc key sway-browser-vnc Tab          # press Tab
charly check vnc key sway-browser-vnc F5           # press F5
charly check vnc key sway-browser-vnc Control_L    # press left Ctrl
```

Valid key names: Return, Escape, Tab, BackSpace, Delete, Home, End, Page_Up, Page_Down, Up, Down, Left, Right, Insert, F1-F12, Shift_L, Shift_R, Control_L, Control_R, Alt_L, Alt_R, Super_L, Super_R, Meta_L, Meta_R, Caps_Lock, space.

### Mouse
```bash
charly check vnc mouse sway-browser-vnc 500 300    # move mouse to (500, 300)
```

### Status
```bash
charly check vnc status sway-browser-vnc
# Output:
# Desktop:    sway
# Resolution: 1920x1080
```

### Password
```bash
charly check vnc passwd sway-browser-vnc              # prompts for password
charly check vnc passwd sway-browser-vnc --generate   # generates random password, prints to stdout
```

Sets up VNC authentication (VeNCrypt/TLS):
1. Stores password in system keyring or config file (depending on `secret_backend` setting) as `vnc.password.<image>`
2. Resolves `$HOME` inside container for absolute config paths
3. Generates self-signed TLS cert+key (valid 3650 days) if not present
4. Generates RSA key in traditional format (`-traditional` flag for OpenSSL 3.x) if not present
5. Writes `~/.config/wayvnc/config` with `enable_auth=true` (wayvnc reads this automatically)
6. Restarts wayvnc supervisord service

After setting a password, all `charly check vnc` commands authenticate transparently via VeNCrypt/TLS.

### Password Resolution Chain

When connecting, password is resolved in this order:
1. `VNC_PASSWORD` environment variable (CI/automation override)
2. System keyring lookup for `vnc.password.<image>-<instance>` (when `secret_backend=auto` or `keyring`)
3. Config file lookup for `vnc.password.<image>-<instance>` (instance-specific)
4. System keyring / config file lookup for `vnc.password.<image>` (image-level)
5. Empty string (no auth — server must allow unauthenticated connections)

```bash
# One-off password override via env
VNC_PASSWORD=secret charly check vnc screenshot sway-browser-vnc out.png

# Set password programmatically (alternative to charly check vnc passwd)
charly settings set vnc.password.sway-browser-vnc mysecret

# Instance-specific password
charly settings set vnc.password.sway-browser-vnc-prod prodpassword
```

Requires `openssl` inside the container for TLS cert and RSA key generation.

### Raw RFB
```bash
charly check vnc rfb sway-browser-vnc key '{"key": 65293, "down": true}'           # raw key event
charly check vnc rfb sway-browser-vnc pointer '{"x": 100, "y": 200, "button": 1}'  # raw pointer
charly check vnc rfb sway-browser-vnc cut-text '{"text": "clipboard"}'              # clipboard
charly check vnc rfb sway-browser-vnc fbupdate-request                              # get dimensions
```

## Differences from CDP Commands

| Aspect | `charly check cdp` (CDP) | `charly check vnc` (RFB) |
|--------|----------------|----------------|
| Protocol | WebSocket JSON | Binary TCP |
| Scope | Browser tabs | Whole desktop |
| Click | CSS selector (viewport-relative) | x,y coordinates (desktop-absolute) |
| Type | CDP key events | Key events (keysyms) |
| Screenshot | Browser page only | Full desktop |
| JavaScript | Yes (check/wait) | No |
| Use case | Web automation | Desktop automation |

Source: `charly/vnc_client.go`, `charly/vnc.go`.

## VNC as Anti-Detection Fallback

Some websites (notably Google sign-in) detect and block CDP-based input. VNC provides a reliable fallback because `charly check vnc type` sends real X11 keysym events through the Wayland compositor — indistinguishable from physical keyboard input.

**CDP + VNC Hybrid Pattern:** Use `charly check cdp click --vnc` for clicking (CDP selector precision + VNC pointer delivery) and `charly check vnc type` for typing credentials:

```bash
# --vnc click: CDP finds element by selector, delivers click via VNC pointer
charly check cdp click my-app $TAB '#identifierId' --vnc
sleep 0.5                                          # let compositor process focus
# VNC type sends real key events through the compositor
charly check vnc type my-app "$GMAIL_USER"
```

**Tested timing:** 500ms sleep between `--vnc` click and VNC type is sufficient. No characters were dropped at this timing during Google sign-in testing.

When to use `--vnc` click and VNC type:
- **`chrome://` pages** (required): CDP mouse events and JS `.click()` are blocked on Chrome's privileged pages (`chrome://intro/`, `chrome://sync-confirmation/`, `chrome://settings/`). `--vnc` is the only way to click.
- Google sign-in or other anti-automation-protected forms
- Sites that validate input event sequences (keyDown/keyPress/input/keyUp)
- Any form where CDP type fails silently (value appears but form doesn't accept it)

**Chrome first-run dialogs:** On fresh profiles, Chrome opens a first-run dialog as a separate window invisible to CDP. Dismiss with `charly check wl sway msg my-app 'focus left'` then `charly check vnc key my-app Return`.

See `/charly-check:cdp` for the full Google sign-in recipe.

## Using CDP Coordinates with VNC

VNC uses desktop-absolute coordinates, while CDP returns viewport-relative coordinates. Use the `--from-cdp` or `--from-sway` flags to explicitly translate:

**`--from-cdp <tab-id>`** — Translates viewport coords to desktop coords via CDP's `window.screenX/screenY`:

```bash
# Get viewport coords from charly check cdp coords, then click via VNC
charly check vnc click my-app 1220 328 --from-cdp $TAB
# Translated viewport (1220, 328) → desktop (1220, 439) via CDP tab ...
```

**`--from-sway <app-id>`** — Translates window-relative coords to desktop coords via sway tree:

```bash
charly check vnc click my-app 500 200 --from-sway google-chrome
# Translated window-relative (500, 200) → desktop (504, 204) via sway app_id=google-chrome
```

Without flags, X and Y are desktop-absolute coordinates (the default, unchanged behavior).

## NVIDIA Headless: VNC Screenshots Work

VNC screenshots work correctly on NVIDIA headless for images using `sway-desktop-vnc` (the standard VNC composition). Two fixes enable this:

1. **Pixman renderer** — `sway-desktop-vnc` forces `WLR_RENDERER=pixman` (software rendering), producing buffers wayvnc can reliably capture
2. **DPMS workaround** — `wayvnc-wrapper` triggers the missing headless power event that wayvnc 0.9.1 waits for before starting capture

Both `charly check vnc screenshot` and `charly check wl screenshot` work on NVIDIA headless:
```bash
charly check vnc screenshot <image> out.png           # VNC screenshot (works with pixman + DPMS fix)
charly check wl screenshot <image> out.png            # Wayland screenshot (grim, always works)
```

## Cross-References

- `/charly-check:check` — parent router; `charly check vnc …` is how every invocation is dispatched.
- `/charly-check:wl` — Wayland-native desktop automation (sibling verb; works on NVIDIA headless).
- `/charly-check:cdp` — Chrome DevTools Protocol automation (sibling verb; same container, different protocol).
- `/charly-check:dbus` — D-Bus calls and desktop notifications (sibling verb under `charly check`).
- `/charly-check:wl` (sway subgroup) — Sway compositor control (window management, workspaces)
- `/charly-core:charly-config` — VNC password storage, `secret_backend` setting, `migrate-secrets` command
- `/charly-core:service` — Managing wayvnc supervisord service
- `/charly-core:deploy` — VNC password setup in deployment workflows
- `/charly-core:shell` — Executing commands inside containers
- `/charly-image:layer` — wayvnc layer configuration (port tcp:5900)

## When to Use This Skill

**MUST be invoked** when the task involves VNC automation, charly check vnc commands, RFB protocol desktop interaction, VNC screenshots, clicking coordinates, or VNC authentication. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Desktop automation. Use for pixel-level interaction when CDP can't reach the element. See also `/charly-check:cdp` (DOM, preferred), `/charly-check:wl` (sway subgroup) (window).
