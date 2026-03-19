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

## Used In Images

Transitive dependency via `chrome-sway` and `sway-desktop` in all desktop images.

## Related Layers

- `/overthink-layers:dbus` -- D-Bus session bus dependency
- `/overthink-layers:chrome-sway` -- Chrome on Sway (depends on sway)
- `/overthink-layers:sway-desktop` -- full desktop composition
- `/overthink-layers:wayvnc` -- VNC access to Sway display
- `/overthink-layers:waybar` -- status bar (depends on sway)

## When to Use This Skill

Use when the user asks about:

- Sway compositor configuration
- Headless Wayland setup
- `WLR_BACKENDS` or Wayland environment variables
- Mesa GPU driver configuration
- Desktop compositor in containers
