---
name: sway
description: |
  Sway Wayland compositor running headless inside containers with Mesa GPU drivers.
  Use when working with Sway, Wayland desktop, or headless compositor setup.
---

# sway -- Headless Wayland compositor

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `dbus` |
| Service | `sway` (supervisord, priority 10) |
| Install files | `tasks:`, `config`, `sway-wrapper` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `WLR_BACKENDS` | `headless` (supervisord env) |
| `WLR_HEADLESS_OUTPUTS` | `1` |
| `WLR_LIBINPUT_NO_DEVICES` | `1` |
| `XDG_RUNTIME_DIR` | `/tmp` |
| `WAYLAND_DISPLAY` | `wayland-0` |
| `EGL_LOG_LEVEL` | `fatal` |
| `XKB_DEFAULT_LAYOUT` | `us` |

### Keyboard Configuration

Keyboard layout defaults to US. Override with `env_accepts` vars:

| Variable | Default | Purpose |
|----------|---------|---------|
| `XKB_DEFAULT_LAYOUT` | `us` | Keyboard layout (us, de, fr, gb, etc.) |
| `XKB_DEFAULT_VARIANT` | (empty) | Layout variant (dvorak, nodeadkeys, etc.) |

```bash
ov config sway-desktop -e XKB_DEFAULT_LAYOUT=de
```

Sway reads `XKB_DEFAULT_LAYOUT` natively from the environment — no wrapper change needed. See `/ov-layers:labwc` for full XKB var list (MODEL, OPTIONS).

The supervisord `[program:sway]` environment sets `WLR_BACKENDS=headless` — headless output only. The `libinput` backend is **not** included by default because it requires a libseat session which fails in rootless containers.

## Packages

- `sway`, `fuzzel` (RPM)
- `mesa-dri-drivers`, `mesa-vulkan-drivers` (RPM)
- `libglvnd-egl`, `libglvnd-gles`, `egl-wayland` (RPM)
- `wlr-randr` (RPM)

## Usage

```yaml
# image.yml -- typically not used directly; pulled in via chrome-sway
my-desktop:
  layers:
    - sway
```

## XWayland

XWayland is **enabled by default** (`xwayland enable` in the base config). The `xorg-x11-server-Xwayland` package is included in the sway layer. XWayland starts lazily — the Xwayland process only launches when the first X11 client connects (e.g., Steam, Heroic, Qt apps with xcb platform).

X11 tools for interacting with XWayland windows (`xdotool`, `xprop`, `xwininfo`, `import`) are provided by the `/ov-layers:wl-tools` layer.

## Stale IPC Socket Cleanup

Supervisord restarts leave old `/tmp/sway-ipc.1000.<old-pid>.sock` files. If multiple sockets exist, naive discovery (`ls /tmp/sway-ipc.*.sock | head -1`) picks alphabetically -- which selects the smallest PID (oldest = stale socket), causing `ov test wl sway` commands to fail silently.

**Fix**: `sway-wrapper` cleans old sockets before starting Sway. Both `sway-wrapper` and `ov test wl sway` (sway.go) use `ls -t | head -1` (newest modification time first) when discovering the active socket.

**Symptoms of stale socket**: `ov test wl sway` commands fail, resolution stays at 1280x720 (wlr-randr resize fails), Chrome renders at wrong size.

## NVIDIA Headless: Renderer

By default, `sway-wrapper` auto-detects GPU hardware and uses `gles2` on NVIDIA. However, if `WLR_RENDERER` is pre-set (e.g., by a composing layer), the wrapper skips GPU auto-detection entirely and uses the specified renderer.

- **VNC images** (`sway-desktop-vnc`) override to `pixman` (software rendering) — ensures reliable VNC screenshot capture on NVIDIA headless
- **Non-VNC images** (Sunshine, standalone sway) use `gles2` via auto-detection — Chrome gets full GPU acceleration
- `grim` (`ov test wl screenshot`) works with both renderers

## Used In Images

Transitive dependency via `chrome-sway` and `sway-desktop` in all desktop images.

## Related Layers

- `/ov-layers:dbus` -- D-Bus session bus dependency
- `/ov-layers:chrome-sway` -- Chrome on Sway (depends on sway)
- `/ov-layers:sway-desktop` -- full desktop composition (VNC)
- `/ov-layers:wayvnc` -- VNC access to Sway display
- `/ov-layers:waybar` -- status bar (depends on sway)
- `/ov-layers:niri` -- Niri compositor alternative (Smithay-based, built from source)

## Related Commands

- `/ov:wl` — Wayland desktop automation (screenshot, input, windows, clipboard, AT-SPI2)

## When to Use This Skill

Use when the user asks about:

- Sway compositor configuration
- Headless Wayland setup
- `WLR_BACKENDS` or Wayland environment variables
- Mesa GPU driver configuration
- Desktop compositor in containers
