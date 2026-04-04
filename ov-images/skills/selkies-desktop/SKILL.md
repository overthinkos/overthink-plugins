# Image: selkies-desktop

Browser-accessible Wayland desktop streamed via Selkies/pixelflux WebSocket at `https://localhost:3000` (HTTPS with self-signed Traefik certificate).

## Definition

```yaml
selkies-desktop:
  base: fedora-nonfree
  layers:
    - agent-forwarding
    - selkies-desktop
  ports:
    - "3000:3000"
    - "9222:9222"
  platforms:
    - linux/amd64
```

## Base

`fedora-nonfree` — Fedora 43 with RPM Fusion (needed for multimedia codecs).

## Layers

`agent-forwarding` (gnupg + direnv + ssh-client) + `selkies-desktop` metalayer = pipewire + chrome + labwc + waybar-labwc + desktop-fonts + swaync + pavucontrol + wl-tools + wl-screenshot-pixelflux + wl-record-pixelflux + a11y-tools + xterm + tmux + asciinema + fastfetch + selkies

## Ports

| Port | Service |
|------|---------|
| 3000 | Selkies web UI (Traefik HTTPS → static files + WebSocket proxy) |
| 9222 | Chrome DevTools Protocol (CDP, via socat relay) |

## Access

Open `https://localhost:3000` in a browser. Accept the self-signed certificate warning (Traefik auto-generates a cert with SANs for localhost, selkies.localhost, and ov-selkies-desktop). The Selkies dashboard shows the labwc desktop with Chrome and Waybar at the top.

HTTPS is required for the WebCodecs API (`VideoDecoder`) used by the Selkies JS client. From other containers on the same network, use `https://ov-selkies-desktop:3000`.

## Build & Deploy

```bash
ov build selkies-desktop
ov start selkies-desktop
# Open https://localhost:3000 (accept cert warning)
```

## Services

| Service | Priority | Purpose |
|---------|----------|---------|
| `selkies` | 8 | Wayland capture + H.264/Opus streaming via pixelflux |
| `traefik` | 18 | HTTPS reverse proxy on port 3000 |
| `selkies-fileserver` | 19 | Static file server for web UI |
| `labwc` | 12 | Wayland compositor (nested in pixelflux) |
| `waybar` | 15 | Status bar |
| `swaync` | 14 | Notification daemon |
| `pipewire` | 5 | Audio server |
| `dbus` | 2 | D-Bus session bus |
| `relay-9222` | — | socat CDP port relay |

## GPU Support

- **Rendering:** NVIDIA GPU via CDI (`DRINODE=/dev/dri/renderD129`), AMD/Intel via Mesa
- **Encoding:** NVENC detected but currently fails with driver 590.48 (pixelflux compat issue). Falls back to CPU x264enc. VAAPI drivers installed for AMD/Intel.
- **CPU fallback:** x264enc-striped at 60fps (16 parallel stripes) — works well

## Volumes

- `chrome-data` → `~/.chrome-debug` (Chrome profile)
- `selkies-config` → `~/.config/selkies`

## Known Issues

1. **NVENC encoding fails** with NVIDIA driver 590.48 — pixelflux detects GPU, CUDA inits, but encoder init fails. CPU encoding works fine.
2. **Chrome volume permissions** — first deploy may need `podman unshare chown 1000:1000 $(podman volume inspect ov-selkies-desktop-chrome-data --format '{{.Mountpoint}}')`
3. **Audio** — PulseAudio null sinks created by selkies-wrapper. Audio streaming works but may have slight latency over WebSocket.

## Screenshots and Recording

The capture bridge provides `ov wl screenshot` and `ov record` support:

```bash
# Screenshot (works with or without browser connected)
ov wl screenshot selkies-desktop screenshot.png

# Check bridge status
ov shell selkies-desktop -c "pixelflux-screenshot --status"

# Desktop video recording (with audio via PulseAudio)
ov record start selkies-desktop -n demo --mode desktop --audio
# ... interact with desktop ...
ov record stop selkies-desktop -n demo -o demo.mp4
```

The bridge auto-heals: if no valid H.264 frames are available (e.g., after browser disconnect), it reconnects as controller to restart the pipeline.

## Client-Side Interaction (Browser-Based Remote Desktop)

When accessing selkies-desktop from another container (e.g., `sway-browser-vnc`) or an external browser, the Selkies SPA provides full mouse and keyboard passthrough via WebSocket.

### SPA DOM Structure

The SPA renders the remote desktop on a `<canvas id="videoCanvas">` with an invisible input overlay that captures all events:

| Element | z-index | pointer-events | Purpose |
|---------|---------|---------------|---------|
| `div.status-display` | 5 | auto | Status bar (hidden by default) |
| `input#overlayInput` | 3 | auto | Transparent input overlay — captures all mouse + keyboard events |
| `canvas#videoCanvas` | 2 | **none** | H.264 video render surface (WebCodecs VideoDecoder → canvas) |
| `button#playButton` | 10 | auto | Play button (hidden after stream starts) |

**Header controls** (fullscreen, gaming mode) slide in from the left edge (`left: -132px`). Move the mouse to the left edge to reveal them.

### Coordinate Scaling

The SPA maps mouse events from the canvas viewport to the remote desktop with a scaling factor. When the canvas is 1908x950 and the remote desktop runs at a different resolution, there is an empirical **~0.824x / 0.836y** ratio between input coordinates and where the remote cursor lands.

Use `ov cdp spa click` with `--scale` for automatic correction:

```bash
# Click at canvas position (990, 375) with scale correction
ov cdp spa click <client> $TAB 990 375 --scale 0.824,0.836
```

### Keyboard Passthrough

**Recommended:** Use `ov cdp spa` commands for keyboard interaction — they bypass the local compositor and Chrome shortcut handlers:

```bash
# Type text (no double-char issue, bypasses local shortcuts)
ov cdp spa type <client> $TAB "hello world"

# Send modifier combos that reach the REMOTE desktop:
ov cdp spa key-combo <client> $TAB super+e    # Open foot terminal in labwc
ov cdp spa key-combo <client> $TAB ctrl+t     # New tab in remote Chrome
ov cdp spa key-combo <client> $TAB alt+f4     # Close window in labwc

# Send special keys
ov cdp spa key <client> $TAB return
ov cdp spa key <client> $TAB escape
```

**Alternative methods** (limited — local compositor/Chrome may intercept keys):

```bash
ov vnc type <client> "text"     # VNC keysym events (Super key intercepted by local compositor)
ov wl type <client> "text"      # wtype via Wayland (same limitation)
```

The SPA's `onkeydown` handler on `#overlayInput` intercepts events with `stopImmediatePropagation()`, converts to keysyms, and sends via WebSocket to the remote labwc compositor.

### Known Limitations (Browser-Based RD)

1. **Coordinate scaling requires `--scale` flag** — auto-detection not yet implemented. Determine the scale empirically by comparing cursor position with target.
2. **Clipboard permission dialog** — On first connection, Chrome prompts "ov-selkies-desktop:3000 wants to See text and images copied to the clipboard". Click Allow or dismiss via keyboard/CDP (`Browser.grantPermissions`).
3. **Closing the last tab exits Chrome** — If the client browser's last tab (the Selkies tab) is closed, Chrome exits. The client container may need a restart to recover Chrome.

### Session Resilience

The remote labwc desktop and all its applications **survive client disconnection**. When the browser tab is closed or the client container restarts, the selkies capture bridge auto-switches from viewer mode to controller mode. On reconnect, the SPA resumes streaming the same desktop state — all windows, typed text, and application state are preserved.

Chrome remembers the self-signed cert exception in the `chrome-data` volume, so no cert re-prompt on reconnect.

### Inter-Container Access

From another container on the `ov` bridge network:

```bash
# URL: https://ov-selkies-desktop:3000
# TLS cert SAN includes DNS:ov-selkies-desktop
# Verify connectivity:
curl -kso /dev/null -w '%{http_code}' https://ov-selkies-desktop:3000/
# Expected: 200

curl -kso /dev/null -w '%{http_code}' https://ov-selkies-desktop:3000/websockets
# Expected: 426 (WebSocket Upgrade Required — correct, not an error)
```

### Streaming Health Checks

```bash
# From within the client browser tab (via CDP eval):
# Check canvas dimensions (non-zero = stream active)
document.getElementById("videoCanvas").width   // e.g., 1908
document.getElementById("videoCanvas").height  // e.g., 950

# Check secure context (required for WebCodecs)
window.isSecureContext                         // true

# Check decoder availability
typeof VideoDecoder !== "undefined"            // true (H.264)
typeof AudioDecoder !== "undefined"            // true (Opus)
```

## Verification

```bash
ov status selkies-desktop              # All services RUNNING
curl -k https://localhost:3000         # HTTPS 200, Selkies dashboard HTML
ov wl screenshot selkies-desktop t.png # Screenshot via capture bridge
ov cdp status selkies-desktop          # CDP available on port 9222
```
