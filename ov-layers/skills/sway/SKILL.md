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
| `WLR_BACKENDS` | `headless` |
| `WLR_HEADLESS_OUTPUTS` | `1` |
| `WLR_LIBINPUT_NO_DEVICES` | `1` |
| `XDG_RUNTIME_DIR` | `/tmp` |
| `WAYLAND_DISPLAY` | `wayland-0` |
| `EGL_LOG_LEVEL` | `fatal` |

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

## Stale IPC Socket Cleanup

Supervisord restarts leave old `/tmp/sway-ipc.1000.<old-pid>.sock` files. If multiple sockets exist, naive discovery (`ls /tmp/sway-ipc.*.sock | head -1`) picks alphabetically -- which selects the smallest PID (oldest = stale socket), causing `ov sway` commands to fail silently.

**Fix**: `sway-wrapper` cleans old sockets before starting Sway. Both `sway-wrapper` and `ov sway` (sway.go) use `ls -t | head -1` (newest modification time first) when discovering the active socket.

**Symptoms of stale socket**: `ov sway` commands fail, resolution stays at 1280x720 (wlr-randr resize fails), Chrome renders at wrong size.

## NVIDIA Headless: Renderer Selection via SWAY_CAPTURE

On NVIDIA headless systems, the renderer depends on the capture method:

| `SWAY_CAPTURE` | Renderer | Pipeline | Used by |
|----------------|----------|----------|---------|
| unset (default) | `pixman` (CPU) | Software rendering, VNC gray screens | `sway-desktop` (wayvnc) |
| `sunshine` | `gles2` (GPU) | Full GPU pipeline, NVENC encoding | `sway-desktop-sunshine` |

**With wayvnc (default):** wayvnc uses `ext-image-copy-capture` which fails with GPU renderers on NVIDIA headless. Forced to pixman. Chrome falls back to software rendering (chrome-wrapper strips NVIDIA EGL vars on pixman).

**With Sunshine:** Sunshine captures via `wlr-screencopy-unstable-v1` (same as grim) which works with gles2 on NVIDIA. The `SWAY_CAPTURE=sunshine` env var (set by the sunshine layer) tells sway-wrapper to use gles2 + NVIDIA EGL vars, enabling full GPU acceleration.

**Impact on Chrome (pixman mode)**: The pixman compositor has no GPU acceleration. Chrome must NOT have NVIDIA EGL vars (`__EGL_VENDOR_LIBRARY_FILENAMES`, `GBM_BACKEND`, `__GLX_VENDOR_LIBRARY_NAME`) or VAAPI flags set, or it crashes with SIGILL. `--disable-gpu` must NOT be used -- it breaks CDP tab creation. The chrome-wrapper strips these vars automatically on pixman. See `/ov-layers:chrome` for details.

## Used In Images

Transitive dependency via `chrome-sway` and `sway-desktop` in all desktop images.

## Related Layers

- `/ov-layers:dbus` -- D-Bus session bus dependency
- `/ov-layers:chrome-sway` -- Chrome on Sway (depends on sway)
- `/ov-layers:sway-desktop` -- full desktop composition (VNC)
- `/ov-layers:sway-desktop-sunshine` -- GPU-accelerated desktop (Sunshine)
- `/ov-layers:wayvnc` -- VNC access to Sway display
- `/ov-layers:sunshine` -- Sunshine game streaming (Moonlight)
- `/ov-layers:waybar` -- status bar (depends on sway)

## When to Use This Skill

Use when the user asks about:

- Sway compositor configuration
- Headless Wayland setup
- `WLR_BACKENDS` or Wayland environment variables
- Mesa GPU driver configuration
- Desktop compositor in containers
