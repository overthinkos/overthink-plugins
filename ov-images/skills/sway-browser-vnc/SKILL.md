---
name: sway-browser-vnc
description: |
  Minimal Sway desktop with VNC remote access and Chrome browser.
  Use when working with VNC desktop containers or testing the sway-desktop-vnc composition.
---

# sway-browser-vnc

Minimal Sway desktop with VNC (wayvnc on port 5900) and Chrome (CDP on port 9222).

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | sway-desktop-vnc |
| Ports | 5900, 9222 |

## Quick Start

```bash
ov build sway-browser-vnc
ov start sway-browser-vnc
ov vnc status sway-browser-vnc
ov wl screenshot sway-browser-vnc screenshot.png
```

## Recording

Desktop video recording via wf-recorder (included in sway-desktop):

```bash
ov record start sway-browser-vnc -n demo --mode desktop
# ... interact ...
ov record stop sway-browser-vnc -n demo -o demo.mp4
```

## Use as Selkies Remote Desktop Client

sway-browser-vnc can act as a test client for selkies-desktop, simulating how a real user would connect via browser:

```bash
# Both containers must be running on the ov bridge network
ov start sway-browser-vnc
ov start selkies-desktop

# Open selkies URL in sway-browser-vnc's Chrome
ov cdp open sway-browser-vnc "https://ov-selkies-desktop:3000"

# Interact with the remote desktop:
# Mouse: CDP Input.dispatchMouseEvent on the Selkies tab (with ~0.82x coordinate scaling)
# Keyboard: ov vnc type sway-browser-vnc "text" (passthrough via SPA's overlayInput)
# Screenshots: ov vnc screenshot sway-browser-vnc (shows full client desktop with stream)
#              ov cdp screenshot sway-browser-vnc $TAB (shows only the stream content)
```

**Key limitations when using as a client:**
- Super key consumed by sway (can't trigger remote labwc keybinds like Super+e)
- Ctrl+T/W consumed by local Chrome (open/close tabs locally, not remotely)
- Clipboard permission dialog appears on first connection — dismiss with `ov vnc key Return` or CDP `Browser.grantPermissions`

See `/ov-images:selkies-desktop` for detailed SPA interaction documentation.

## Related Images

- `selkies-desktop` — browser-accessible remote desktop (can be accessed from sway-browser-vnc's Chrome)
