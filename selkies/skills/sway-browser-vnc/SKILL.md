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
| Layers | agent-forwarding, sway-desktop-vnc, dbus, charly |
| Ports | 5900, 9222, 9224 |

## Quick Start

```bash
charly box build sway-browser-vnc
charly start sway-browser-vnc
charly status sway-browser-vnc          # Shows all probes: supervisord, cdp, dbus, ov, sway, vnc, wl
charly eval vnc status sway-browser-vnc
charly eval wl screenshot sway-browser-vnc screenshot.png
```

## D-Bus and Notification Support

This image includes `dbus` and `ov` layers, enabling:
- `charly eval dbus notify` — native Go D-Bus notifications via in-container charly binary
- `charly eval dbus list/call/introspect` — full D-Bus interaction
- `charly cmd` — single command execution with desktop notification on completion
- `charly tmux cmd` — tmux command sending with notification
- `charly status` — supervisord, dbus, and charly probes

```bash
charly eval dbus notify sway-browser-vnc "Build Complete" "Image built successfully"
charly cmd sway-browser-vnc "make test"    # Notifies on completion
charly eval dbus list sway-browser-vnc          # List all D-Bus services
```

The notification daemon (swaync) is included via the sway-desktop metalayer.

## Overlays

Fullscreen overlays for recording (via wl-overlay, included in sway-desktop):

```bash
charly eval wl overlay show sway-browser-vnc --type text --text "INTRO" --name title
charly eval wl overlay show sway-browser-vnc --type lower-third --text "Speaker" --subtitle "Role" --name lt
charly eval wl overlay show sway-browser-vnc --type countdown --seconds 3 --name cd
charly eval wl overlay show sway-browser-vnc --type highlight --region "430,290,510,50" --name hl
charly eval wl overlay show sway-browser-vnc --type fade --color black --name outro
charly eval wl overlay hide sway-browser-vnc --all
```

All overlay types render with true RGBA compositor transparency. See `/charly-eval:wl-overlay` for the full recording workflow.

## Recording

Desktop video recording via wf-recorder (included in sway-desktop):

```bash
charly eval record start sway-browser-vnc -n demo --mode desktop
# ... interact ...
charly eval record stop sway-browser-vnc -n demo -o demo.mp4
```

## Use as Selkies Remote Desktop Client

sway-browser-vnc can act as a test client for selkies-desktop, simulating how a real user would connect via browser:

```bash
# Both containers must be running on the charly bridge network
charly start sway-browser-vnc
charly start selkies-desktop

# Open selkies URL in sway-browser-vnc's Chrome
charly eval cdp open sway-browser-vnc "https://charly-selkies-desktop:3000"

# Interact with the remote desktop:
# Mouse: CDP Input.dispatchMouseEvent on the Selkies tab (with ~0.82x coordinate scaling)
# Keyboard: charly eval vnc type sway-browser-vnc "text" (passthrough via SPA's overlayInput)
# Screenshots: charly eval vnc screenshot sway-browser-vnc (shows full client desktop with stream)
#              charly eval cdp screenshot sway-browser-vnc $TAB (shows only the stream content)
```

**Key limitations when using as a client:**
- Super key consumed by sway (can't trigger remote labwc keybinds like Super+e)
- Ctrl+T/W consumed by local Chrome (open/close tabs locally, not remotely)
- Clipboard permission dialog appears on first connection — dismiss with `charly eval vnc key Return` or CDP `Browser.grantPermissions`

See `/charly-selkies:selkies-labwc` for detailed SPA interaction documentation.

## Test Coverage

Latest `charly eval live sway-browser-vnc` run: **84 passed, 0 failed, 1 skipped**
(`chrome-devtools-mcp-port` references `${HOST_PORT:9224}` which isn't
mapped here — correct skip behavior).

Covers all 19 transitive layers: wayvnc (VNC), sway (compositor),
chrome-sway, xdg-portal, waybar, swaync, pavucontrol, thunar,
xfce4-terminal, pipewire, wl-screenshot-grim, wl-overlay, wf-recorder,
desktop-fonts, fastfetch, asciinema, tmux, dbus, ov. Deploy-scope: VNC
port 5900 reachable, Chrome CDP on port 9250→9222 with `/json/version`
200. Image-scope: sway + wayvnc both RUNNING under supervisord.

## Related Skills

- `/charly-selkies:sway-desktop-vnc`, `/charly-selkies:sway`, `/charly-selkies:wayvnc`,
  `/charly-selkies:chrome-sway`, `/charly-selkies:xdg-portal`, `/charly-infrastructure:dbus-layer`,
  `/charly-tools:charly`, `/charly-distros:agent-forwarding`
- `/charly-eval:eval` — declarative testing framework (parent router for `charly eval cdp|wl|dbus|vnc|mcp`)
- `/charly-eval:vnc` — VNC automation on this image
- `/charly-eval:cdp` — Chrome automation (CDP on host port 9250)
- `/charly-eval:wl` — Wayland input/windows/clipboard (sway subgroup for compositor control)
- `/charly-eval:dbus` — D-Bus notifications via in-container `ov` binary
- `/charly-build:ov-mcp-cmd` — the image inherits 2 deploy-scope `mcp:` checks from the `chrome-devtools-mcp` layer (ping + list-tools asserting `navigate_page`/`take_screenshot`). `charly eval live sway-browser-vnc --filter mcp` runs them; note the **port-publishing gotcha** — if your `deploy.yml` has an explicit `port:` override that predates `chrome-devtools-mcp`, port 9224 may not be published. See `/charly-build:ov-mcp-cmd` for the fix.

## Related Images

- `selkies-desktop` — browser-accessible remote desktop (can be accessed from sway-browser-vnc's Chrome)

## Related

- `/charly-image:image` — image family umbrella (`image:` entries in `charly.yml`, build/validate/inspect/list)
- `/charly-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
