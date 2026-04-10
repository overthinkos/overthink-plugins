---
name: chrome
description: |
  Google Chrome with DevTools on port 9222, Chrome DevTools MCP on port 9224, and browser-open helper.
  Use when working with Chrome, CDP, browser automation, or DevTools Protocol.
---

# chrome -- Google Chrome with DevTools

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord` |
| Composed layers | `chrome-devtools-mcp` |
| Ports | 9222 (CDP, via cdp-proxy), 9224 (Chrome DevTools MCP, via chrome-devtools-mcp) |
| CDP Proxy | `cdp-proxy` on 0.0.0.0:9222 → Chrome on 127.0.0.1:9223 |
| Volumes | `chrome-data` -> `~/.chrome-debug` |
| Security | `shm_size: 1g` |
| Install files | `layer.yml`, `root.yml`, `user.yml`, `cdp-proxy`, `chrome-wrapper`, `chrome-restart`, `browser-open` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `CHROME_FLAGS` | `--ozone-platform=wayland --enable-features=UseOzonePlatform,VaapiVideoDecodeLinuxGL,VaapiIgnoreDriverChecks` |
| `BROWSER` | `browser-open` |

PATH additions: `~/.local/bin`

## Environment Provides

| Variable | Template Value |
|----------|---------------|
| `BROWSER_CDP_URL` | `http://{{.ContainerName}}:9222` |

Pod-aware: same-container consumers receive `http://localhost:9222`, cross-container consumers receive `http://ov-<image>:9222`. Consumed by hermes (`env_accepts: BROWSER_CDP_URL`) for shared browser automation.

## MCP Provides

| Name | URL Template | Transport |
|------|-------------|-----------|
| `chrome-devtools` | `http://{{.ContainerName}}:9224/mcp` | http |

Provided by the auto-included `chrome-devtools-mcp` sub-layer (29 Chrome DevTools tools via mcp-proxy). Consumed by hermes (`mcp_accepts: chrome-devtools`) for MCP-based browser inspection and automation. See `/ov-layers:chrome-devtools-mcp` for tool list and architecture.

When deploying with `-i <instance>`, the MCP server name is automatically disambiguated to `chrome-devtools-<instance>` (e.g., `chrome-devtools-31.58.9.4`). See `/ov:config` for MCP name disambiguation details.

## Environment Accepts (Proxy)

| Variable | Description |
|----------|-------------|
| `HTTP_PROXY` | HTTP proxy URL for Chrome (e.g., `http://proxy:8080`) |
| `HTTPS_PROXY` | HTTPS proxy URL for Chrome (e.g., `http://proxy:8080`) |
| `NO_PROXY` | Comma-separated bypass list (e.g., `localhost,127.0.0.1,.internal`) |

Chrome does NOT natively respect `HTTP_PROXY`/`HTTPS_PROXY` environment variables. The `chrome-wrapper` script translates them to Chrome CLI flags at launch:

- Both set: `--proxy-server=http=<HTTP_PROXY>;https=<HTTPS_PROXY>`
- Only one set: `--proxy-server=<value>` (applies to all protocols)
- `NO_PROXY` → `--proxy-bypass-list=<list>` (only added when a proxy is configured)

Uppercase takes precedence over lowercase (`HTTP_PROXY` over `http_proxy`). The `chrome-x11-wrapper` has identical proxy translation logic.

```bash
# Deploy with proxy
ov config selkies-desktop -e HTTP_PROXY=http://proxy:8080 -e "NO_PROXY=localhost,127.0.0.1"

# Per-instance proxy (separate from base deployment)
ov config selkies-desktop -i proxy \
  -e HTTP_PROXY=http://31.58.9.4:6077 \
  -e HTTPS_PROXY=http://31.58.9.4:6077 \
  -e "NO_PROXY=localhost,127.0.0.1" \
  -p 3001:3000 -p 9232:9222
```

## Packages

RPM (system): `liberation-fonts`, `vulkan-loader`, `mesa-dri-drivers`, `alsa-lib`, `at-spi2-core`, `cups-libs`, `gtk3`, `libdrm`, `libxkbcommon`, `nss`, `procps-ng`, `iproute`, `libva-utils`
RPM (root.yml): `google-chrome-stable` (from Google's direct RPM)

## Usage

```yaml
# images.yml
my-browser:
  layers:
    - chrome
```

Usually used via the `chrome-sway` or `sway-desktop` composition layers rather than directly.

## Google Sign-In

Web sign-in at `accounts.google.com` works via CDP + VNC hybrid automation (see `/ov:cdp` for the full recipe). All clicks use `ov cdp click --vnc` (CDP selector targeting + VNC pointer delivery), and all text input uses `ov vnc type` (real OS-level keysym events). Sign-in cookies persist in the `chrome-data` volume (`~/.chrome-debug`), surviving container restarts. Use `ov remove <image> --purge` to clear for a fresh start — just rebuilding the image does not reset volumes.

**App Passwords (required for automation):** Google accounts with 2FA (now mandatory for most accounts) require a 16-character [App Password](https://myaccount.google.com/apppasswords). App Passwords bypass all verification challenges and 2FA prompts. Set `GMAIL_PASSWORD` to the App Password in `.env`.

**Windows spoofing effectiveness:** Tested with App Password — no CAPTCHA or verification challenges were triggered. The three-level identity spoofing (User-Agent header, navigator.platform JS, Sec-CH-UA-Platform headers) successfully prevents Google from flagging the sign-in as suspicious.

**Fresh profile first-run:** A fresh `chrome-data` volume triggers Chrome's first-run flow: a separate dialog window ("Make Google Chrome the default browser") + `chrome://intro/` page. The dialog is invisible to CDP and must be dismissed via sway focus + VNC Return key before proceeding.

**Chrome Sync:** Fully supported. After sign-in, Chrome navigates to `chrome://sync-confirmation/` (a `chrome://` page, not a native dialog). Click `#confirmButton` ("Yes, I'm in") with `--vnc` to enable sync. Sync state persists in the `chrome-data` volume.

## Chrome Wrapper

All Chrome launches go through `chrome-wrapper` (`~/.local/bin/chrome-wrapper`), which adds CDP flags (`--remote-debugging-port=9223`), Windows User-Agent spoofing, and GPU detection. A symlink `~/.local/bin/google-chrome-stable -> chrome-wrapper` ensures Chrome always uses the wrapper regardless of how it's launched (sway exec, desktop file, app menu, or Chrome's self-restart after close/crash). The wrapper calls `/opt/google/chrome/chrome` directly to avoid recursion through the symlink.

### GPU Detection

The chrome-wrapper detects the sway renderer by reading Sway's `/proc/<pid>/environ` for `WLR_RENDERER`. On NVIDIA with gles2 (the default), Chrome gets full GPU acceleration. The wrapper sets NVIDIA EGL vars and VAAPI flags automatically.

If `WLR_RENDERER=pixman` is detected (e.g., explicit override via `WLR_RENDERER` env var), the wrapper strips NVIDIA EGL vars and VAAPI flags so Chrome falls back to software rendering naturally. `--disable-gpu` must NOT be used — it breaks CDP tab creation.

The detection uses `pgrep -x sway` (exact process name match — NOT `pgrep -f` which matches sway-wrapper and sway-autotile).

## CDP Proxy

Chrome 146+ rejects HTTP requests to the DevTools API when the `Host` header contains a non-localhost, non-IP hostname (e.g., `ov-selkies-desktop:9222`). This breaks cross-container CDP access via `env_provides: BROWSER_CDP_URL`.

**Architecture:** A `cdp-proxy` Python supervisord service listens on `0.0.0.0:9222` and forwards requests to Chrome on `127.0.0.1:9223`. Chrome binds only to the internal loopback address.

**Host header rewrite:** Incoming requests with container hostnames (e.g., `Host: ov-selkies-desktop:9222`) are rewritten to `Host: localhost:9223` before forwarding to Chrome, satisfying Chrome's localhost-only check.

**Response URL rewrite:** Chrome's DevTools HTTP API returns WebSocket URLs containing `ws://localhost:9223/...` in JSON responses. The proxy rewrites these to `ws://<client-host>:9222/...` (using the original `Host` header from the client request) so that cross-container WebSocket connections route back through the proxy.

**Content-Length correction:** After URL rewriting, response body sizes change. The proxy recalculates and corrects the `Content-Length` header to match the rewritten body.

The chrome layer depends on `supervisord` (not `socat`) to run cdp-proxy as a managed service.

### ShaderCache / GpuCache Cleanup

The wrapper cleans `ShaderCache` and `GpuCache` directories from the Chrome profile on startup. Corrupted caches (from `pkill -9` or hard container stops) cause secondary SIGILL crashes even with correct GPU flags. Cleaning on every launch prevents this.

## browser-open On-Demand Chrome Launch

The `browser-open` helper (set as `BROWSER` env var) handles three states:

1. **Chrome running + CDP responsive**: Reuses the existing instance, opens the URL in a new tab.
2. **Chrome running + CDP unresponsive**: Waits for CDP to become available (avoids double-launching).
3. **Chrome not running**: Launches Chrome via `swaymsg exec chrome-wrapper` with SWAYSOCK discovery (glob on `/tmp/sway-ipc.*.sock`, newest first).

Uses `pgrep` to detect Chrome process state before deciding whether to launch. This is used by OAuth flows (`BROWSER=browser-open` causes OAuth URLs to auto-open in Chrome via CDP).

## chrome-restart

The `chrome-restart` script kills any running Chrome and relaunches it via `chrome-wrapper` with all proxy, CDP, and user-agent settings intact. Used by waybar's `custom/chrome` button (see `/ov-layers:waybar`) for one-click Chrome restart.

**Compositor detection:** Tries `swaymsg exec` first (sway — needs IPC context for proper window management), falls back to direct `chrome-wrapper &` (labwc). Uses the same SWAYSOCK discovery pattern as `browser-open` and `waybar-wrapper` (`ls -t /tmp/sway-ipc.*.sock`).

**Kill sequence:** SIGTERM first, then SIGKILL after 1 second for stubborn processes. Matches Chrome by `chrome.*remote-debugging-port` pattern (same as `browser-open`).

```bash
# Manual usage (inside container)
chrome-restart

# Via ov shell
ov shell selkies-desktop -i 45.39.130.177 -c "chrome-restart"
```

## Windows Platform Spoofing

Chrome launches with a Windows User-Agent and loads the `chrome-windows` extension to present a consistent Windows identity:

- **User-Agent header:** Overridden to `Mozilla/5.0 (Windows NT 10.0; Win64; x64)...` via `--user-agent` flag
- **navigator.platform:** Spoofed to `"Win32"` via content script (`chrome-windows/content.js`)
- **navigator.userAgentData.platform:** Spoofed to `"Windows"`
- **Sec-CH-UA-Platform header:** Set to `"Windows"` via declarativeNetRequest rules (`chrome-windows/rules.json`)

This reduces the chance of Google flagging the sign-in as suspicious (Linux + automation fingerprint).

## NVIDIA Headless Notes

Most images use `gles2` on NVIDIA headless — Chrome gets full GPU acceleration. VNC images (`sway-desktop-vnc`) use `pixman` instead — Chrome falls back to software rendering (chrome-wrapper auto-detects `WLR_RENDERER=pixman` and strips NVIDIA EGL vars). No special handling needed — the chrome-wrapper adapts automatically to both renderers.

## Chrome 147+ CDP Changes

Chrome 147 introduced a breaking change to the CDP HTTP API:

- **`/json/new` requires PUT** — `GET /json/new?<url>` returns `405 Method Not Allowed`. Use `PUT /json/new?<url>` instead. This affects programmatic tab creation via curl or HTTP clients.
- **Host header validation** (since Chrome 146) — Non-localhost Host headers are rejected. The `cdp-proxy` rewrites headers to satisfy this check (see CDP Proxy section above).

```bash
# Chrome 147+: create new tab (PUT required)
curl -s -X PUT "http://localhost:9222/json/new?https://example.com"

# List tabs (GET still works)
curl -s "http://localhost:9222/json/list"
```

## CDP Diagnostics

`ov cdp` commands now show diagnostics on connection failure: checks Chrome process, cdp-proxy status, and port binding. Hints use `ov wl sway exec <image> chrome-wrapper` (not `ov shell` with bare `swaymsg`) for manual Chrome restart.

## Used In Images

- Part of `chrome-sway` / `sway-desktop` composition (used in `openclaw-sway-browser`, `openclaw-ollama-sway-browser`)

## Related Layers

- `/ov-layers:chrome-devtools-mcp` — Chrome DevTools MCP server (auto-included via `layers:`)
- `/ov-layers:supervisord` — required dependency for cdp-proxy service
- `/ov-layers:hermes` — consumes `BROWSER_CDP_URL` via `env_accepts` and `chrome-devtools` via `mcp_accepts`
- `/ov-layers:selkies-desktop` — desktop metalayer composing chrome with labwc, pipewire, waybar, etc.

## Related Commands

- `/ov:cdp` — Chrome DevTools Protocol automation (click, type, eval, screenshot)
- `/ov:shell` — Interactive shell to access Chrome
- `/ov:vnc` — VNC automation (used with `--vnc` flag on `ov cdp click`)
- `/ov:wl` — Wayland automation (used with `--wl` flag on `ov cdp click`)
- `/ov:config` — Proxy deployment, `normalizeNoProxy()` auto-conversion, `sep:"none"` env handling

## When to Use This Skill

Use when the user asks about:

- Google Chrome in containers
- Chrome DevTools Protocol (CDP) on port 9222
- Browser automation or `browser-open`
- The `chrome` layer, `CHROME_FLAGS`, or `shm_size`
