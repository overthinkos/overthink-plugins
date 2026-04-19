# Image: selkies-desktop

Browser-accessible Wayland desktop streamed via Selkies/pixelflux WebSocket at `https://localhost:3000` (HTTPS with self-signed Traefik certificate).

## Definition

```yaml
selkies-desktop:
  base: fedora-nonfree
  layers:
    - agent-forwarding
    - selkies-desktop
    - dbus
    - ov
  ports:
    - "3000:3000"
    - "9222:9222"
    - "9224:9224"
  platforms:
    - linux/amd64
```

Tunnel config is in `deploy.yml` (not image.yml): `tunnel: {provider: tailscale, private: all}`. See `/ov:deploy`.

## Base

`fedora-nonfree` — Fedora 43 with RPM Fusion (needed for multimedia codecs).

## Layers

`agent-forwarding` (gnupg + direnv + ssh-client) + `selkies-desktop` metalayer (pipewire + chrome + labwc + waybar-labwc + desktop-fonts + swaync + pavucontrol + wl-tools + wl-screenshot-pixelflux + wl-overlay + wl-record-pixelflux + a11y-tools + xterm + tmux + asciinema + fastfetch + selkies) + `dbus` + `ov`

## Ports

| Port | Service |
|------|---------|
| 3000 | Selkies web UI (Traefik HTTPS → static files + WebSocket proxy) |
| 9222 | Chrome DevTools Protocol (CDP, via cdp-proxy) |
| 9224 | Chrome DevTools MCP (Streamable HTTP, via chrome-devtools-mcp) |

## Access

Open `https://localhost:3000` in a browser. Accept the self-signed certificate warning (Traefik auto-generates a cert with SANs for localhost, selkies.localhost, and ov-selkies-desktop). The Selkies dashboard shows the labwc desktop with Chrome and Waybar at the top.

HTTPS is required for the WebCodecs API (`VideoDecoder`) used by the Selkies JS client. From other containers on the same network, use `https://ov-selkies-desktop:3000`.

## Quick Start

```bash
ov image build selkies-desktop
ov config selkies-desktop
ov start selkies-desktop
# Access: https://localhost:3000 (accept cert warning)
ov test wl screenshot selkies-desktop screenshot.png
```

## Keyboard Configuration

Override the default US keyboard layout via environment variables:

```bash
ov config selkies-desktop -e XKB_DEFAULT_LAYOUT=de                              # German QWERTZ
ov config selkies-desktop -e XKB_DEFAULT_LAYOUT=fr -e XKB_DEFAULT_VARIANT=nodeadkeys  # French, no dead keys
```

| Variable | Default | Purpose |
|----------|---------|---------|
| `XKB_DEFAULT_LAYOUT` | `us` | Keyboard layout |
| `XKB_DEFAULT_VARIANT` | (empty) | Layout variant |
| `XKB_DEFAULT_MODEL` | `pc105` | Keyboard model |
| `XKB_DEFAULT_OPTIONS` | (empty) | XKB options |

All level 0/1 characters (ö, ä, ü, ß, =, ?) and AltGr characters (@, €, \\, ~) work for any layout. See `/ov-layers:labwc` for details and `/ov-layers:selkies` for the input pipeline.

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
| `cdp-proxy` | — | CDP reverse proxy (Host header + URL rewrite for Chrome 146+) |

## GPU Support

- **Rendering:** NVIDIA GPU via CDI, AMD/Intel via Mesa. `DRINODE`/`DRI_NODE` auto-detected at runtime by `ov config` (from first `/dev/dri/renderD*`)
- **VAAPI encoding (AMD):** Hardware H264 encoding via VAAPI — requires correct DRINODE (auto-detected). Wrong DRINODE causes CPU fallback → swapchain buffer exhaustion → stream flickering
- **NVENC (NVIDIA):** Detected but currently fails with driver 590.48 (pixelflux compat issue). Falls back to CPU x264enc
- **CPU fallback:** x264enc-striped at 60fps (16 parallel stripes) — works but may cause flickering at high resolutions due to compositor buffer pressure

## Volumes

- `chrome-data` → `~/.chrome-debug` (Chrome profile)
- `selkies-config` → `~/.config/selkies`

## Known Issues

1. **NVENC encoding fails** with NVIDIA driver 590.48 — pixelflux detects GPU, CUDA inits, but encoder init fails. CPU encoding works fine.
2. **Chrome volume permissions** — first deploy may need `podman unshare chown 1000:1000 $(podman volume inspect ov-selkies-desktop-chrome-data --format '{{.Mountpoint}}')`
3. **Audio** — PulseAudio null sinks created by selkies-wrapper. Audio streaming works but may have slight latency over WebSocket.

## Screenshots and Recording

The capture bridge provides `ov test wl screenshot` and `ov record` support:

```bash
# Screenshot (works with or without browser connected)
ov test wl screenshot selkies-desktop screenshot.png

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

Use `ov test cdp spa click` with `--scale` for automatic correction:

```bash
# Click at canvas position (990, 375) with scale correction
ov test cdp spa click <client> $TAB 990 375 --scale 0.824,0.836
```

### Keyboard Passthrough

**Recommended:** Use `ov test cdp spa` commands for keyboard interaction — they bypass the local compositor and Chrome shortcut handlers:

```bash
# Type text (no double-char issue, bypasses local shortcuts)
ov test cdp spa type <client> $TAB "hello world"

# Send modifier combos that reach the REMOTE desktop:
ov test cdp spa key-combo <client> $TAB super+e    # Open foot terminal in labwc
ov test cdp spa key-combo <client> $TAB ctrl+t     # New tab in remote Chrome
ov test cdp spa key-combo <client> $TAB alt+f4     # Close window in labwc

# Send special keys
ov test cdp spa key <client> $TAB return
ov test cdp spa key <client> $TAB escape
```

**Alternative methods** (limited — local compositor/Chrome may intercept keys):

```bash
ov test vnc type <client> "text"     # VNC keysym events (Super key intercepted by local compositor)
ov test wl type <client> "text"      # wtype via Wayland (same limitation)
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

## Deploy with Tailscale Exit Node

Route all outbound internet traffic (Chrome browsing) through a Tailscale exit node while keeping the desktop accessible on the host's tailnet and the ov bridge.

### Prerequisites

1. A Tailscale auth key for the sidecar's tailnet (can be a different tailnet from the host)
2. An exit node device advertising on that tailnet (approved in admin console)

### Setup

```bash
# Store auth key
ov secrets gpg set TS_AUTHKEY tskey-auth-xxxxxxxxxxxx
ov secrets gpg set TS_EXIT_NODE 100.80.254.4    # exit node's Tailscale IP

# Deploy with sidecar
ov config selkies-desktop --sidecar tailscale \
  -e TS_HOSTNAME=selkies-desktop \
  -e "TS_EXTRA_ARGS=--exit-node=${TS_EXIT_NODE} --exit-node-allow-lan-access"
ov start selkies-desktop

# First time: set exit node inside sidecar (persists in state volume)
podman exec ov-selkies-desktop-tailscale \
  tailscale set --exit-node=100.80.254.4 --exit-node-allow-lan-access
```

### Verify Dual Networking

```bash
# Exit node routing — shows exit node's public IP, not host's
podman exec ov-selkies-desktop curl -s ifconfig.me

# Bridge connectivity — other ov containers reachable
podman exec ov-selkies-desktop getent hosts ov-ollama

# Host tailnet — accessible via host's tailscale serve
curl -sk https://o.armadillo-quail.ts.net:3000/ | head -1

# Sidecar status
podman exec ov-selkies-desktop-tailscale tailscale status
```

### Architecture

The pod has dual networking: `Network=ov` (bridge for container-to-container) + `tailscale0` (tun interface for exit node). `--exit-node-allow-lan-access` adds `throw 10.89.0.0/24` to exempt bridge traffic. The deploy.yml `tunnel: tailscale` config generates `ExecStartPost=tailscale serve` to expose ports 3000+9222 on the host's tailnet independently.

**Known issues:**
- `TS_DEBUG_FIREWALL_MODE=nftables` is required (iptables-legacy fails in rootless podman) — built into the sidecar template
- `ShmSize=1g` is required for Chrome — automatically propagated to the pod via `PodmanArgs=--shm-size`
- Exit node device must be **approved** on the sidecar's tailnet admin console
- First-time exit node: use `tailscale set --exit-node` inside sidecar (persists in state volume for restarts)

See `/ov:sidecar` for full sidecar documentation.

## Multi-Instance Proxy Deployment

Deploy multiple instances with different HTTP proxies for IP-diverse browsing. Each instance gets unique host ports and its own MCP server (`chrome-devtools-<instance>`).

```bash
# Deploy 3 instances with different proxies (ports 3001-3003, CDP 9231-9233)
ov config selkies-desktop -i 45.39.130.21 \
  -e HTTP_PROXY=http://45.39.130.21:6753 \
  -e HTTPS_PROXY=http://45.39.130.21:6753 \
  -e 'NO_PROXY=localhost,127.0.0.1' \
  -p 3001:3000 -p 9231:9222

# Propagate MCP to hermes, then start
ov config hermes --update-all
ov start selkies-desktop -i 45.39.130.21

# Verify proxy IP
ov test cdp open selkies-desktop -i 45.39.130.21 "https://httpbin.org/ip"
```

**Tailscale access (no sidecar needed):** The deploy.yml `tunnel: tailscale` config generates `tailscale serve` commands for host-mapped ports. All instances are accessible via the host's Tailscale IP on their respective ports (`https://<host>:3001`, etc.). Use sidecars only when per-instance exit node routing is needed.

**MCP auto-disambiguation:** Each instance provides `chrome-devtools-<instance>` MCP server. Consumers (hermes) receive all instances in `OV_MCP_SERVERS` JSON after `--update-all`.

See `/ov:config` for `--update-all` propagation, `/ov-layers:chrome` for `env_accepts` (HTTP_PROXY/HTTPS_PROXY/NO_PROXY).

## Build Pipeline Note

The selkies-desktop image now compiles `pixelflux_wayland` from source in the
fedora-builder stage. This is because pixelflux's upstream wheel does not include the
**dmabuf cache cleanup fix** (`renderer.cleanup_texture_cache()` per frame) that
prevents a Wayland compositor shmem leak under sustained heavy streaming. The patch is
applied at build time via inline source patching in `layers/selkies/build.sh`. See
`/ov-layers:selkies` (Patched pixelflux build pipeline) for the full pipeline and the
diagnostic recipe that found the leak.

## Related Images

- `/ov-images:selkies-desktop-nvidia` — GPU-accelerated variant with NVIDIA CUDA toolkit (base: nvidia instead of fedora-nonfree)
- `/ov-images:sway-browser-vnc` — VNC-based alternative using Sway compositor instead of Selkies/labwc streaming

## Verification

```bash
ov status selkies-desktop              # All services RUNNING
curl -k https://localhost:3000         # HTTPS 200, Selkies dashboard HTML
ov test wl screenshot selkies-desktop t.png # Screenshot via capture bridge
ov test cdp status selkies-desktop          # CDP available on port 9222
```

## Test Coverage

Latest `ov test selkies-desktop` run: **91 passed, 0 failed, 0 skipped**
— the largest test suite in the project. Covers all 21 transitive
layers (selkies, chrome, sshd, chrome-devtools-mcp primary; labwc,
waybar-labwc, pipewire, swaync, pavucontrol, wl-tools, wl-*-pixelflux,
a11y-tools, xterm, desktop-fonts, asciinema, fastfetch, tmux
secondary).

Deploy-scope: ports 3000 (HTTPS selkies), 9222 (Chrome CDP), 9224
(chrome-devtools-mcp), 2222 (sshd) all reachable via
`127.0.0.1:${HOST_PORT:N}`. `/json/version` returns 200 with
`webSocketDebuggerUrl`. All primary services RUNNING under supervisord
(labwc, selkies, traefik, chrome via event-listener handoff, sshd).

Note: the sshd layer uses `sudo -n -l` rather than `file:` existence
for `/etc/sudoers.d/ov-user` because it's root-only (`/ov:test` Gotcha #10).

## Related Skills

- `/ov-layers:selkies-desktop` (metalayer), `/ov-layers:selkies`,
  `/ov-layers:chrome`, `/ov-layers:labwc`, `/ov-layers:sshd`,
  `/ov-layers:chrome-devtools-mcp`, `/ov-layers:pipewire`
- `/ov:test` — declarative testing framework + testing gotchas
- `/ov:cdp`, `/ov:wl` — desktop automation on this image
- `/ov:config` — deploy setup (tunnel, port remapping, instances)
- `/ov:mcp` — the image bundles `chrome-devtools-mcp` (transitively via the chrome metalayer), so 2 deploy-scope `mcp:` checks (`ping`, `list-tools`) run against its MCP server on port 9224. `ov test selkies-desktop --filter mcp` runs them; `ov test mcp list-tools selkies-desktop` enumerates the 29 chrome-devtools tools ad-hoc.
