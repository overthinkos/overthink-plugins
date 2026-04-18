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
| Layers | agent-forwarding, sway-desktop-vnc, dbus, ov |
| Ports | 5900, 9222, 9224 |

## Quick Start

```bash
ov image build sway-browser-vnc
ov start sway-browser-vnc
ov status sway-browser-vnc          # Shows all probes: supervisord, cdp, dbus, ov, sway, vnc, wl
ov vnc status sway-browser-vnc
ov wl screenshot sway-browser-vnc screenshot.png
```

## D-Bus and Notification Support

This image includes `dbus` and `ov` layers, enabling:
- `ov dbus notify` — native Go D-Bus notifications via in-container ov binary
- `ov dbus list/call/introspect` — full D-Bus interaction
- `ov cmd` — single command execution with desktop notification on completion
- `ov tmux cmd` — tmux command sending with notification
- `ov status` — supervisord, dbus, and ov probes

```bash
ov dbus notify sway-browser-vnc "Build Complete" "Image built successfully"
ov cmd sway-browser-vnc "make test"    # Notifies on completion
ov dbus list sway-browser-vnc          # List all D-Bus services
```

The notification daemon (swaync) is included via the sway-desktop metalayer.

## Overlays

Fullscreen overlays for recording (via wl-overlay, included in sway-desktop):

```bash
ov wl overlay show sway-browser-vnc --type text --text "INTRO" --name title
ov wl overlay show sway-browser-vnc --type lower-third --text "Speaker" --subtitle "Role" --name lt
ov wl overlay show sway-browser-vnc --type countdown --seconds 3 --name cd
ov wl overlay show sway-browser-vnc --type highlight --region "430,290,510,50" --name hl
ov wl overlay show sway-browser-vnc --type fade --color black --name outro
ov wl overlay hide sway-browser-vnc --all
```

All overlay types render with true RGBA compositor transparency. See `/ov:wl-overlay` for the full recording workflow.

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

## Test Coverage

Latest `ov test sway-browser-vnc` run: **84 passed, 0 failed, 1 skipped**
(`chrome-devtools-mcp-port` references `${HOST_PORT:9224}` which isn't
mapped here — correct skip behavior).

Covers all 19 transitive layers: wayvnc (VNC), sway (compositor),
chrome-sway, xdg-portal, waybar, swaync, pavucontrol, thunar,
xfce4-terminal, pipewire, wl-screenshot-grim, wl-overlay, wf-recorder,
desktop-fonts, fastfetch, asciinema, tmux, dbus, ov. Deploy-scope: VNC
port 5900 reachable, Chrome CDP on port 9250→9222 with `/json/version`
200. Image-scope: sway + wayvnc both RUNNING under supervisord.

## Related Skills

- `/ov-layers:sway-desktop-vnc`, `/ov-layers:sway`, `/ov-layers:wayvnc`,
  `/ov-layers:chrome-sway`, `/ov-layers:xdg-portal`
- `/ov:test` — declarative testing framework
- `/ov:vnc` — VNC automation on this image
- `/ov:cdp` — Chrome automation (CDP on host port 9250)

## Related Images

- `selkies-desktop` — browser-accessible remote desktop (can be accessed from sway-browser-vnc's Chrome)
