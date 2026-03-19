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

## Used In Images

- Part of `chrome-sway` / `sway-desktop` composition (used in `openclaw-sway-browser`, `openclaw-ollama-sway-browser`)

## Related Layers

- `/overthink-layers:socat` -- required dependency for DevTools port relay

## When to Use This Skill

Use when the user asks about:

- Google Chrome in containers
- Chrome DevTools Protocol (CDP) on port 9222
- Browser automation or `browser-open`
- The `chrome` layer, `CHROME_FLAGS`, or `shm_size`
