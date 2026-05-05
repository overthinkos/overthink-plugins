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
# image.yml
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

| Component | kwin-desktop | sway-desktop |
|-----------|-------------|--------------|
| Compositor | KWin (KDE) | Sway (wlroots) |
| Portal | xdg-portal-kde | xdg-portal (wlr) |
| Terminal | Konsole | xfce4-terminal |
| File manager | Dolphin | Thunar |
| Status bar | None | Waybar |
| Chrome binding | chrome-kwin | chrome-sway |

## Related Layers

- `/ov-selkies:kwin` -- KWin compositor
- `/ov-selkies:sway-desktop` -- Sway variant

## Used In Images

Not used in any current image definition. Available as a metalayer composition but no image references it yet.

## When to Use This Skill

Use when working with KWin-based desktop containers or comparing desktop compositions.

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov eval live`)
