---
name: vnc
description: |
  VNC automation: ov vnc commands for RFB protocol desktop interaction.
  Use when capturing VNC screenshots, clicking at coordinates, typing text,
  sending key events, or setting up VNC authentication for deployments.
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
```

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
1. Stores password in `ov config` as `vnc.password.<image>` (host side, for automatic client auth)
2. Resolves `$HOME` inside container for absolute config paths
3. Generates self-signed TLS cert+key (valid 3650 days) if not present
4. Generates RSA key in traditional format (`-traditional` flag for OpenSSL 3.x) if not present
5. Writes `~/.config/wayvnc/config` with `enable_auth=true` (wayvnc reads this automatically)
6. Restarts wayvnc supervisord service

After setting a password, all `ov vnc` commands authenticate transparently via VeNCrypt/TLS.

### Password Resolution Chain

When connecting, password is resolved in this order:
1. `VNC_PASSWORD` environment variable (CI/automation override)
2. `ov config get vnc.password.<image>-<instance>` (instance-specific)
3. `ov config get vnc.password.<image>` (image-level)
4. Empty string (no auth — server must allow unauthenticated connections)

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
| Click | CSS selector | x,y coordinates |
| Type | DOM manipulation | Key events (keysyms) |
| Screenshot | Browser page only | Full desktop |
| JavaScript | Yes (eval/wait) | No |
| Use case | Web automation | Desktop automation |

Source: `ov/vnc_client.go`, `ov/vnc.go`.

## Cross-references

- `/ov:cdp` — Chrome DevTools Protocol automation (same container, different protocol)
- `/ov:sway` — Sway compositor control (window management, workspaces)
- `/ov:config` — VNC password storage (`vnc.password.<image>` config keys)
- `/ov:service` — Managing wayvnc supervisord service
- `/ov:deploy` — VNC password setup in deployment workflows
- `/ov:shell` — Executing commands inside containers
- `/ov:layer` — wayvnc layer configuration (port tcp:5900)

## When to Use This Skill

Use when the user asks about:

- `ov vnc` commands (screenshot, click, type, key, mouse, status, passwd, rfb)
- VNC desktop automation and pixel-level interaction
- VNC password management and VeNCrypt/TLS setup
- Coordinate-based clicking or typing
- "How do I interact with the desktop via VNC?"

**Workflow position:** Desktop automation. Use for pixel-level interaction when CDP can't reach the element. See also `/ov:cdp` (DOM, preferred), `/ov:sway` (window).
