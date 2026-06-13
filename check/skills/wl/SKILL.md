---
name: wl
description: |
  MUST be invoked before any work involving: Wayland desktop automation — `charly check wl` commands (screenshots, click/type/scroll/drag, window management, clipboard, resolution control, AT-SPI2 introspection, window geometry), nested `wl sway` / `wl overlay` subcommands, or `wl:` declarative verbs inside `check:` blocks. Covers sway-desktop and selkies-desktop image automation on sway, labwc, AND KWin (KDE Plasma) — window management routes to wlrctl on wlroots and kdotool on KWin.
---

# WL - Wayland Desktop Automation

## Overview

`charly check wl` is the unified desktop automation command for wlroots compositors (sway, labwc) **and KWin (KDE Plasma)**. It provides screenshots, input (click, type, key combos, scroll, drag), window management, clipboard, resolution control, accessibility introspection (AT-SPI2), and window geometry queries. Works on sway-desktop, selkies-desktop (labwc), and selkies-kde-desktop (KWin) images.

**Per-compositor routing.** `detectCompositor` picks the backend per method. On wlroots: `wlrctl` (pointer + `wlrctl toplevel` window management), `wlr-randr` (resolution). On **KWin**: window management (toplevel/windows/focus/close/fullscreen/minimize/geometry) via **`kdotool`** (KWin scripting), keyboard via `wtype`, clipboard via `wl-clipboard`, screenshot via `pixelflux`; `status` reports `compositor: kwin`. KWin **pointer** (click/double-click/mouse/scroll/drag) and **resolution** have no host-safe backend on the headless nested KWin and return a clear "unsupported on KWin" error (not a hang) — see the Compositor Compatibility table below.

### Also as a declarative verb

Every `charly check wl <method>` (including nested `wl overlay <method>` and `wl sway <method>`) is authorable as a `wl:` verb inside a `check:` block. Nested subcommands are hyphenated in YAML: `wl: overlay-show`, `wl: sway-tree`, `wl: sway-workspaces`. Method-specific fields (`x`, `y`, `text`, `key`, `combo`, `target`, `action`, `artifact`) are siblings of the verb line. See `/charly-check:check` for the full method allowlist. Example: `- wl: screenshot\n  artifact: /tmp/desktop.png\n  artifact_min_bytes: 10000`.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Screenshot | `charly check wl screenshot <image> [file]` | Capture desktop as PNG via grim |
| Click | `charly check wl click <image> <x> <y>` | Click at absolute coordinates via wlrctl |
| Double-click | `charly check wl double-click <image> <x> <y>` | Double-click with configurable delay |
| Type text | `charly check wl type <image> <text>` | Send keyboard input via wtype |
| Send key | `charly check wl key <image> <key-name>` | Press a named key via wtype |
| Key combo | `charly check wl key-combo <image> <keys>` | Send key combination (ctrl+c, alt+tab) |
| Move mouse | `charly check wl mouse <image> <x> <y>` | Move pointer to absolute coordinates |
| Scroll | `charly check wl scroll <image> <x> <y> <dir>` | Scroll at coordinates (up/down/left/right) |
| Drag | `charly check wl drag <image> <x1> <y1> <x2> <y2>` | Drag between coordinates (experimental) |
| List windows | `charly check wl windows <image>` | List windows (wlrctl toplevel, xdotool fallback) |
| List toplevel | `charly check wl toplevel <image>` | List Wayland toplevel windows via wlrctl |
| Focus window | `charly check wl focus <image> <title>` | Focus window (wlrctl toplevel, xdotool fallback) |
| Close window | `charly check wl close <image> <title>` | Close window via wlrctl toplevel |
| Fullscreen | `charly check wl fullscreen <image> <title>` | Toggle fullscreen via wlrctl toplevel |
| Minimize | `charly check wl minimize <image> <title>` | Toggle minimize via wlrctl toplevel |
| Launch app | `charly check wl exec <image> <command>` | Launch application in container |
| Resolution | `charly check wl resolution <image> <WxH>` | Set output resolution via wlr-randr |
| Clipboard | `charly check wl clipboard <image> get/set/clear` | Read/write Wayland clipboard |
| Window props | `charly check wl xprop <image> [target]` | Query X11 window properties |
| Window rect | `charly check wl geometry <image> <target>` | Get window position/size as JSON |
| A11y tree | `charly check wl atspi <image> tree` | Dump accessibility tree as JSON |
| A11y find | `charly check wl atspi <image> find <query>` | Find elements by name/role |
| A11y click | `charly check wl atspi <image> click <query>` | Click element by name/role |
| Status | `charly check wl status <image>` | Check all tool availability |
| Overlay show | `charly check wl overlay show <image> --type text --text "Hello"` | Show recording overlay (see `/charly-check:wl-overlay`) |
| Overlay hide | `charly check wl overlay hide <image> --all` | Remove all overlays |

## Compositor Compatibility

Backend availability per compositor (`charly check wl` routes to the available one):

| Tool | Protocol | sway | labwc (selkies) | KWin (KDE Plasma) |
|------|----------|------|-----------------|-------------------|
| grim | wlr-screencopy | YES | NO (nested compositor) | NO |
| pixelflux-screenshot | pixelflux API | NO | YES | YES |
| wtype | zwp_virtual_keyboard_v1 | YES | YES | YES (KWin implements it) |
| wlrctl pointer | wlr-virtual-pointer | YES | YES | NO — pointer unsupported on KWin |
| wlrctl toplevel | wlr-foreign-toplevel-management | YES | YES | NO — window mgmt via `kdotool` |
| kdotool | KWin scripting (D-Bus) | NO | NO | YES (toplevel/focus/close/fullscreen/minimize/geometry) |
| wlr-randr | wlr-output-management | YES | YES | NO — resolution unsupported on KWin |
| wl-copy/paste | wlr-data-control | YES | YES | YES (KWin implements it) |
| xdotool | X11 (XWayland) | YES | YES (on-demand) | YES (on-demand) |
| swaymsg | i3 IPC | YES | NO | NO |

**KWin (selkies-kde-desktop) notes.** Window management routes to `kdotool` (the
`kde-shell` layer); keyboard/clipboard/screenshot reuse `wtype`/`wl-clipboard`/
`pixelflux`. Pointer (click/double-click/mouse/scroll/drag) and resolution have no
host-safe backend on the headless nested KWin (`org_kde_kwin_fake_input` is removed
in KWin 6, the RemoteDesktop portal is approval-gated, `/dev/uinput` leaks into the
host, `kscreen-doctor` hangs) and return a clear "unsupported on KWin" error.

**Coordinate translation flags:**
- `--from-cdp` — Works on all compositors (uses `window.screenX/Y`)
- `--from-sway` — Sway only (uses `swaymsg -t get_tree`)
- `--from-x11` — Works on all compositors with XWayland

## Architecture

```
CLI command -> resolveContainer (engine + container name)
           -> exec sh -c "export WAYLAND_DISPLAY=... && <tool command>"
           -> capture stdout (screenshot) or run silently (input)
```

Uses `exec` into the container. All tools use native Wayland protocols — no daemon, no `/dev/uinput`, no VNC server required.

## Requirements

- Container must include `wl-tools` layer (wtype, wlrctl, wl-clipboard, wlr-randr, xdotool, ydotool)
- For screenshots: `wl-screenshot-grim` (sway) or `wl-screenshot-pixelflux` (selkies)
- Container must have a running Wayland compositor (sway, labwc, etc.)
- For AT-SPI2: `a11y-tools` layer (python3-pyatspi, python3-gobject) + `dbus` layer
- For XWayland: an X11 app like `xterm` must be running to trigger XWayland start on labwc
- Included in `sway-desktop` and `selkies-desktop` metalayers

## Commands

### Key Combo
```bash
charly check wl key-combo my-app ctrl+c          # Ctrl+C
charly check wl key-combo my-app alt+tab         # Alt+Tab
charly check wl key-combo my-app ctrl+shift+t    # Ctrl+Shift+T
charly check wl key-combo my-app super+l         # Super+L (lock)
```

Modifiers: `ctrl`/`control`, `alt`, `shift`, `super`/`win`/`logo`, `meta`. Uses `wtype -M`.

### Scroll
```bash
charly check wl scroll my-app 960 540 down              # scroll down 3 steps at center
charly check wl scroll my-app 960 540 up --amount 10    # scroll up 10 steps
```

Uses xdotool click 4/5/6/7 (X11 scroll buttons) for XWayland windows. Falls back to wtype Page_Up/Page_Down.

### Drag (Experimental)
```bash
charly check wl drag my-app 100 100 500 500                # drag left to right
charly check wl drag my-app 100 100 500 500 --duration 500  # slower drag (500ms)
```

Requires XWayland (uses `xdotool mousemove + mousedown/mouseup`).

### Window Management (wlrctl toplevel)
```bash
charly check wl toplevel my-app                  # list all windows
charly check wl focus my-app "Chrome"            # focus by title
charly check wl close my-app "Chrome"            # close by title
charly check wl fullscreen my-app "Chrome"       # toggle fullscreen
charly check wl minimize my-app "Chrome"         # toggle minimize
charly check wl exec my-app foot                 # launch terminal
```

### Resolution
```bash
charly check wl resolution my-app 1920x1080              # auto-detect output
charly check wl resolution my-app 2560x1440 -o WL-1      # specific output
```

### Clipboard
```bash
charly check wl clipboard my-app get                   # read clipboard
charly check wl clipboard my-app set "hello"           # write clipboard
charly check wl clipboard my-app clear                 # clear clipboard
charly check wl clipboard my-app get --primary         # read primary selection
```

### Window Geometry
```bash
charly check wl geometry my-app "Chrome"    # returns JSON: {"x":0,"y":0,"width":1920,"height":1080}
charly check wl xprop my-app                # active window properties
charly check wl xprop my-app "Chrome"      # specific window properties
```

### AT-SPI2 Accessibility
```bash
charly check wl atspi my-app tree                    # dump full accessibility tree
charly check wl atspi my-app find "Save"             # find elements named "Save"
charly check wl atspi my-app find "button"           # find elements with role "button"
charly check wl atspi my-app find "Save:button"      # find by name AND role
charly check wl atspi my-app click "Save:button"     # click element by name/role
```

Requires `a11y-tools` layer. Chrome needs `--force-renderer-accessibility` flag.

### CDP → WL Bridge

Use `charly check cdp click --wl` to find elements by CSS selector in Chrome and deliver clicks via wlrctl (critical for selkies-desktop which has no VNC):

```bash
charly check cdp click selkies-desktop $TAB '#submit-button' --wl
```

## Sway-Specific Commands (`charly check wl sway`)

Sway IPC commands are grouped under `charly check wl sway <subcommand>`. These require a sway compositor and use swaymsg. They will error on labwc.

```bash
charly check wl sway tree <image>              # Get sway window tree (JSON)
charly check wl sway msg <image> 'focus left'  # Run any swaymsg command
charly check wl sway workspaces <image>        # List workspaces (JSON)
charly check wl sway outputs <image>           # List outputs (JSON)
charly check wl sway focus <image> left        # Focus by direction or [criteria]
charly check wl sway move <image> right        # Move focused window
charly check wl sway resize <image> width 10px # Resize focused window
charly check wl sway kill <image>              # Close focused window
charly check wl sway floating <image>          # Toggle floating
charly check wl sway layout <image> tabbed     # Set layout mode
charly check wl sway workspace <image> 2       # Switch workspace
charly check wl sway reload <image>            # Reload sway config
```

## Differences from VNC

| Aspect | `charly check wl` | `charly check vnc` |
|--------|---------|----------|
| Compositors | All wlroots (+ sway subgroup) | Requires wayvnc |
| Transport | exec into container | TCP port 5900 |
| Window mgmt | wlrctl toplevel + sway IPC | No |
| Clipboard | wl-copy/paste | rfb cut-text |
| Remote access | No | Yes (TCP) |
| NVIDIA headless | Works | Works (pixman + DPMS fix) |

Source: `charly/wl.go`.

## Cross-References

- `/charly-check:check` — parent router; `charly check wl …` is how every invocation is dispatched.
- `/charly-check:vnc` — VNC/RFB protocol alternative (sibling verb; TCP-based, works remotely).
- `/charly-check:cdp` — Chrome DevTools Protocol (sibling verb; DOM-level interaction, `--wl` flag for click, `axtree` for accessibility).
- `/charly-check:dbus` — D-Bus calls and desktop notifications (sibling verb under `charly check`).
- `/charly-selkies:wl-tools` — Compositor-agnostic tools (wtype, wlrctl, wl-clipboard, wlr-randr, xdotool, ydotool)
- `/charly-check:wl-overlay` — Fullscreen overlays for recordings (title cards, lower-thirds, countdowns, highlights, fades)
- `/charly-selkies:wl-overlay-layer` — Overlay layer (gtk4-layer-shell, python3-gobject)
- `/charly-selkies:wl-screenshot-grim` — Screenshot layer for sway (grim, wlr-screencopy)
- `/charly-selkies:wl-screenshot-pixelflux` — Screenshot layer for selkies (pixelflux rendering pipeline)
- `/charly-selkies:a11y-tools` — AT-SPI2 accessibility (python3-pyatspi, python3-gobject)
- `/charly-selkies:xterm` — X11 terminal for XWayland testing
- `/charly-selkies:sway-desktop` — Desktop metalayer (wl-tools + wl-screenshot-grim)
- `/charly-selkies:selkies-desktop-layer` — Desktop metalayer (wl-tools + wl-screenshot-pixelflux + a11y-tools + xterm)
