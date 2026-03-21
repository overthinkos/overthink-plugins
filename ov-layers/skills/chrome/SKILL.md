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

Web sign-in at `accounts.google.com` works via CDP automation (see `/ov:cdp` for the full recipe). Sign-in cookies persist in the `chrome-data` volume (`~/.chrome-debug`), surviving container restarts.

**App Passwords:** Google accounts with 2FA enabled (now mandatory for most accounts) require a 16-character [App Password](https://myaccount.google.com/apppasswords) instead of the regular password. Set `GMAIL_PASSWORD` to the App Password in `.env`.

**Chrome Sync:** Fully supported. After signing in to Google and clicking "Turn on Sync" in `chrome://settings/syncSetup`, passwords, bookmarks, and extensions sync across devices. Sync state persists in the `chrome-data` volume.

## Chrome Wrapper

All Chrome launches go through `chrome-wrapper` (`~/.local/bin/chrome-wrapper`), which adds CDP flags (`--remote-debugging-port=9222`), Windows User-Agent spoofing, and GPU detection. A symlink `~/.local/bin/google-chrome-stable -> chrome-wrapper` ensures Chrome always uses the wrapper regardless of how it's launched (sway exec, desktop file, app menu, or Chrome's self-restart after close/crash). The wrapper calls `/opt/google/chrome/chrome` directly to avoid recursion through the symlink.

## Windows Platform Spoofing

Chrome launches with a Windows User-Agent and loads the `chrome-windows` extension to present a consistent Windows identity:

- **User-Agent header:** Overridden to `Mozilla/5.0 (Windows NT 10.0; Win64; x64)...` via `--user-agent` flag
- **navigator.platform:** Spoofed to `"Win32"` via content script (`chrome-windows/content.js`)
- **navigator.userAgentData.platform:** Spoofed to `"Windows"`
- **Sec-CH-UA-Platform header:** Set to `"Windows"` via declarativeNetRequest rules (`chrome-windows/rules.json`)

This reduces the chance of Google flagging the sign-in as suspicious (Linux + automation fingerprint).

## Used In Images

- Part of `chrome-sway` / `sway-desktop` composition (used in `openclaw-sway-browser`, `openclaw-ollama-sway-browser`)

## Related Layers

- `/ov-layers:socat` -- required dependency for DevTools port relay

## When to Use This Skill

Use when the user asks about:

- Google Chrome in containers
- Chrome DevTools Protocol (CDP) on port 9222
- Browser automation or `browser-open`
- The `chrome` layer, `CHROME_FLAGS`, or `shm_size`
