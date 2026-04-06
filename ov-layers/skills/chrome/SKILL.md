---
name: chrome
description: |
  Google Chrome with DevTools on port 9222, port relay, and browser-open helper.
  Use when working with Chrome, CDP, browser automation, or DevTools Protocol.
---

# chrome -- Google Chrome with DevTools

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `socat` |
| Ports | 9222 |
| Port relay | 9222 (eth0 -> loopback) |
| Volumes | `chrome-data` -> `~/.chrome-debug` |
| Security | `shm_size: 1g` |
| Install files | `layer.yml`, `root.yml`, `user.yml` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `CHROME_FLAGS` | `--ozone-platform=wayland --enable-features=UseOzonePlatform,VaapiVideoDecodeLinuxGL,VaapiIgnoreDriverChecks` |
| `BROWSER` | `browser-open` |

PATH additions: `~/.local/bin`

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

All Chrome launches go through `chrome-wrapper` (`~/.local/bin/chrome-wrapper`), which adds CDP flags (`--remote-debugging-port=9222`), Windows User-Agent spoofing, and GPU detection. A symlink `~/.local/bin/google-chrome-stable -> chrome-wrapper` ensures Chrome always uses the wrapper regardless of how it's launched (sway exec, desktop file, app menu, or Chrome's self-restart after close/crash). The wrapper calls `/opt/google/chrome/chrome` directly to avoid recursion through the symlink.

### GPU Detection

The chrome-wrapper detects the sway renderer by reading Sway's `/proc/<pid>/environ` for `WLR_RENDERER`. On NVIDIA with gles2 (the default), Chrome gets full GPU acceleration. The wrapper sets NVIDIA EGL vars and VAAPI flags automatically.

If `WLR_RENDERER=pixman` is detected (e.g., explicit override via `WLR_RENDERER` env var), the wrapper strips NVIDIA EGL vars and VAAPI flags so Chrome falls back to software rendering naturally. `--disable-gpu` must NOT be used — it breaks CDP tab creation.

The detection uses `pgrep -x sway` (exact process name match — NOT `pgrep -f` which matches sway-wrapper and sway-autotile).

### ShaderCache / GpuCache Cleanup

The wrapper cleans `ShaderCache` and `GpuCache` directories from the Chrome profile on startup. Corrupted caches (from `pkill -9` or hard container stops) cause secondary SIGILL crashes even with correct GPU flags. Cleaning on every launch prevents this.

## browser-open On-Demand Chrome Launch

The `browser-open` helper (set as `BROWSER` env var) handles three states:

1. **Chrome running + CDP responsive**: Reuses the existing instance, opens the URL in a new tab.
2. **Chrome running + CDP unresponsive**: Waits for CDP to become available (avoids double-launching).
3. **Chrome not running**: Launches Chrome via `swaymsg exec chrome-wrapper` with SWAYSOCK discovery (glob on `/tmp/sway-ipc.*.sock`, newest first).

Uses `pgrep` to detect Chrome process state before deciding whether to launch. This is used by OAuth flows (`BROWSER=browser-open` causes OAuth URLs to auto-open in Chrome via CDP).

## Windows Platform Spoofing

Chrome launches with a Windows User-Agent and loads the `chrome-windows` extension to present a consistent Windows identity:

- **User-Agent header:** Overridden to `Mozilla/5.0 (Windows NT 10.0; Win64; x64)...` via `--user-agent` flag
- **navigator.platform:** Spoofed to `"Win32"` via content script (`chrome-windows/content.js`)
- **navigator.userAgentData.platform:** Spoofed to `"Windows"`
- **Sec-CH-UA-Platform header:** Set to `"Windows"` via declarativeNetRequest rules (`chrome-windows/rules.json`)

This reduces the chance of Google flagging the sign-in as suspicious (Linux + automation fingerprint).

## NVIDIA Headless Notes

Most images use `gles2` on NVIDIA headless — Chrome gets full GPU acceleration. VNC images (`sway-desktop-vnc`) use `pixman` instead — Chrome falls back to software rendering (chrome-wrapper auto-detects `WLR_RENDERER=pixman` and strips NVIDIA EGL vars). No special handling needed — the chrome-wrapper adapts automatically to both renderers.

## CDP Diagnostics

`ov cdp` commands now show diagnostics on connection failure: checks Chrome process, relay status, and port binding. Hints use `ov wl sway exec <image> chrome-wrapper` (not `ov shell` with bare `swaymsg`) for manual Chrome restart.

## Used In Images

- Part of `chrome-sway` / `sway-desktop` composition (used in `openclaw-sway-browser`, `openclaw-ollama-sway-browser`)

## Related Layers

- `/ov-layers:socat` -- required dependency for DevTools port relay

## Related Commands

- `/ov:cdp` — Chrome DevTools Protocol automation (click, type, eval, screenshot)
- `/ov:shell` — Interactive shell to access Chrome

## When to Use This Skill

Use when the user asks about:

- Google Chrome in containers
- Chrome DevTools Protocol (CDP) on port 9222
- Browser automation or `browser-open`
- The `chrome` layer, `CHROME_FLAGS`, or `shm_size`
