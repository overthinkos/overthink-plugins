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

Uses `github.com/kward/go-vnc` for RFB protocol (RFC 6143).

## Requirements

- Container must include the `wayvnc` layer (port tcp:5900)
- Container must be running (`ov start` or `ov enable`)
- Wayland compositor must be active (sway, cage, or niri)

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

Sets up VNC authentication:
1. Stores password in `ov config` (host side, for automatic auth)
2. Generates TLS cert/key + RSA key inside container
3. Writes wayvnc config file with auth enabled
4. Restarts wayvnc service

After setting a password, all `ov vnc` commands authenticate automatically.

### Raw RFB
```bash
ov vnc rfb openclaw-sway-browser key '{"key": 65293, "down": true}'           # raw key event
ov vnc rfb openclaw-sway-browser pointer '{"x": 100, "y": 200, "button": 1}'  # raw pointer
ov vnc rfb openclaw-sway-browser cut-text '{"text": "clipboard"}'              # clipboard
ov vnc rfb openclaw-sway-browser fbupdate-request                              # get dimensions
```

## Differences from Browser Commands

| Aspect | `ov browser` (CDP) | `ov vnc` (RFB) |
|--------|-------------------|----------------|
| Protocol | WebSocket JSON | Binary TCP |
| Scope | Browser tabs | Whole desktop |
| Click | CSS selector | x,y coordinates |
| Type | DOM manipulation | Key events (keysyms) |
| Screenshot | Browser page only | Full desktop |
| JavaScript | Yes (eval/wait) | No |
| Use case | Web automation | Desktop automation |

## Cross-references

- `/overthink:browser` — Chrome browser automation (same container, different protocol)
- `/overthink:service` — Managing wayvnc supervisord service
- `/overthink:shell` — Executing commands inside containers
- `/overthink:layer` — wayvnc layer configuration (port tcp:5900)
