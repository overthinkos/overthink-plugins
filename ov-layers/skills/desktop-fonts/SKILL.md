---
name: desktop-fonts
description: |
  JetBrains Mono and Nerd Fonts for desktop containers.
  Use when working with font configuration or desktop text rendering.
---

# desktop-fonts -- JetBrains Mono + Nerd Fonts

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | none |
| Service | none |
| Install files | `layer.yml` only |

## Packages

- `jetbrains-mono-fonts` (RPM) -- monospace coding font
- `liberation-fonts` (RPM) -- serif/sans/mono web fonts (Chrome rendering fallback)
- `nerd-fonts` (RPM, via `che/nerd-fonts` COPR) -- icon font (Symbols Nerd Font)

## Usage

Included in desktop metalayers. Not typically added directly:

```yaml
# Already part of sway-desktop and selkies-desktop metalayers
```

## Notes

- The `che/nerd-fonts` COPR is enabled before install and disabled after (declarative `copr:` field)
- Provides `Symbols Nerd Font` and `Symbols Nerd Font Mono` for waybar icons and swaync
- JetBrains Mono is the primary font for waybar and swaync Catppuccin Mocha theme

## Used In

- `/ov-layers:sway-desktop` -- via metalayer composition
- `/ov-layers:selkies-desktop` -- via metalayer composition

## Related Layers

- `/ov-layers:waybar` -- uses JetBrains Mono + Symbols Nerd Font
- `/ov-layers:waybar-labwc` -- same fonts
- `/ov-layers:swaync` -- uses JetBrains Mono

## When to Use This Skill

Use when the user asks about:

- Font configuration in desktop containers
- Nerd Font icons in waybar or terminal
- JetBrains Mono font availability
