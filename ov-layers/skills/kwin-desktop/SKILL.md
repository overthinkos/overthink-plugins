---
name: kwin-desktop
description: |
  KDE desktop composition with KWin, PipeWire, XDG Portal, Chrome, Konsole, and Dolphin.
  Use when working with KWin desktop containers.
---

# kwin-desktop -- KDE desktop metalayer

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `pipewire`, `xdg-portal-kde`, `chrome-kwin`, `kwin-apps` |
| Install files | none (pure composition) |

## Usage

```yaml
# images.yml
my-kwin-desktop:
  base: nvidia
  layers:
    - kwin-desktop
```

## Included Layers

- `pipewire` -- Audio/media server (PulseAudio compat)
- `xdg-portal-kde` -- KDE portal backend (ScreenCast, RemoteDesktop)
- `chrome-kwin` -- Chrome browser with DevTools
- `kwin-apps` -- Konsole terminal + Dolphin file manager

## Comparison with Other Desktops

| Component | kwin-desktop | sway-desktop | niri-desktop |
|-----------|-------------|--------------|--------------|
| Compositor | KWin (KDE) | Sway (wlroots) | Niri (Smithay) |
| Portal | xdg-portal-kde | xdg-portal (wlr) | xdg-portal-niri |
| Terminal | Konsole | xfce4-terminal | xfce4-terminal |
| File manager | Dolphin | Thunar | Thunar |
| Status bar | None | Waybar | None |
| Chrome binding | chrome-kwin | chrome-sway | chrome-niri |

## Related Layers

- `/ov-layers:kwin` -- KWin compositor
- `/ov-layers:sway-desktop` -- Sway variant
- `/ov-layers:niri-desktop` -- Niri variant

## Used In Images

Not used in any current image definition. Available as a metalayer composition but no image references it yet.

## When to Use This Skill

Use when working with KWin-based desktop containers or comparing desktop compositions.
