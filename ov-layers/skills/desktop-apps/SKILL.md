---
name: desktop-apps
description: |
  Desktop applications including Chromium, VLC, KeePassXC, btop, cockpit, and zsh.
  Use when working with GUI applications or desktop environment setup.
---

# desktop-apps -- Desktop GUI applications

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |

## Packages

RPM: `btop`, `chromium`, `cloud-init`, `cockpit`, `keepassxc`, `transmission`, `vlc`, `zsh`

## Usage

```yaml
# image.yml
my-desktop:
  layers:
    - desktop-apps
```

## Used In Images

- `bazzite-ai` (disabled)

## Related Layers
- `/ov-layers:copr-desktop` — Sibling COPR desktop packages (Ghostty, Nerd Fonts)
- `/ov-layers:dev-tools` — Sibling CLI utilities bundled in the same bootc image
- `/ov-layers:desktop-fonts` — Sibling font configuration for desktop containers

## Related Commands
- `/ov:build` — Build the bootc image including these desktop packages
- `/ov:vm` — Run the bootc image as a VM to test the desktop apps

## When to Use This Skill

Use when the user asks about:

- Desktop applications in containers
- Chromium, VLC, KeePassXC, or cockpit
- The `desktop-apps` layer or its packages

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
