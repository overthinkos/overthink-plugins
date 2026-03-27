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
| Install files | `root.yml`, `user.yml`, `config`, `sway-wrapper` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `WLR_BACKENDS` | `headless` (supervisord env) |
| `WLR_HEADLESS_OUTPUTS` | `1` |
| `WLR_LIBINPUT_NO_DEVICES` | `1` |
| `XDG_RUNTIME_DIR` | `/tmp` |
| `WAYLAND_DISPLAY` | `wayland-0` |
| `EGL_LOG_LEVEL` | `fatal` |

The supervisord `[program:sway]` environment sets `WLR_BACKENDS=headless` — headless output only. The `libinput` backend is **not** included by default because it requires a libseat session which fails in rootless containers.

## Packages

- `sway`, `fuzzel`, `mako` (RPM)
- `mesa-dri-drivers`, `mesa-vulkan-drivers` (RPM)
- `libglvnd-egl`, `libglvnd-gles`, `egl-wayland` (RPM)
- `wlr-randr` (RPM)

## Usage

```yaml
# images.yml -- typically not used directly; pulled in via chrome-sway
my-desktop:
  layers:
    - sway
```

## XWayland

XWayland is **enabled by default** (`xwayland enable` in the base config). The `xorg-x11-server-Xwayland` package is included in the sway layer. XWayland starts lazily — the Xwayland process only launches when the first X11 client connects (e.g., Steam, Heroic, Qt apps with xcb platform).

X11 tools for interacting with XWayland windows (`xdotool`, `xprop`, `xwininfo`, `import`) are provided by the `/ov-layers:wl-tools` layer.

## Stale IPC Socket Cleanup

Supervisord restarts leave old `/tmp/sway-ipc.1000.<old-pid>.sock` files. If multiple sockets exist, naive discovery (`ls /tmp/sway-ipc.*.sock | head -1`) picks alphabetically -- which selects the smallest PID (oldest = stale socket), causing `ov wl sway` commands to fail silently.

**Fix**: `sway-wrapper` cleans old sockets before starting Sway. Both `sway-wrapper` and `ov wl sway` (sway.go) use `ls -t | head -1` (newest modification time first) when discovering the active socket.

**Symptoms of stale socket**: `ov wl sway` commands fail, resolution stays at 1280x720 (wlr-randr resize fails), Chrome renders at wrong size.

## NVIDIA Headless: Renderer

All images use `gles2` on NVIDIA (hardware auto-detect in sway-wrapper). No renderer overrides or application-specific conditionals.

- `grim` (`ov wl screenshot`) captures real desktop content via `wlr-screencopy` — works with gles2
- `wayvnc` screenshots are gray (upstream `ext-image-copy-capture` bug) — use `ov wl` instead
- Chrome gets full GPU acceleration with gles2

## Used In Images

Transitive dependency via `chrome-sway` and `sway-desktop` in all desktop images.

## Related Layers

- `/ov-layers:dbus` -- D-Bus session bus dependency
- `/ov-layers:chrome-sway` -- Chrome on Sway (depends on sway)
- `/ov-layers:sway-desktop` -- full desktop composition (VNC)
- `/ov-layers:wayvnc` -- VNC access to Sway display
- `/ov-layers:waybar` -- status bar (depends on sway)
- `/ov-layers:niri` -- Niri compositor alternative (Smithay-based, built from source)

## When to Use This Skill

Use when the user asks about:

- Sway compositor configuration
- Headless Wayland setup
- `WLR_BACKENDS` or Wayland environment variables
- Mesa GPU driver configuration
- Desktop compositor in containers
