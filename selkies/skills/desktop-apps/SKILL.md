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
| Install files | `candy.yml` (packages only) |

## Packages

RPM: `btop`, `chromium`, `cloud-init`, `cockpit`, `keepassxc`, `transmission`, `vlc`, `zsh`

## Usage

```yaml
# box.yml
my-desktop:
  layers:
    - desktop-apps
```

## Used In Images

- `bazzite`

## Related Layers
- `/charly-distros:copr-desktop` — Sibling COPR desktop packages (Ghostty, Nerd Fonts)
- `/charly-coder:dev-tools` — Sibling CLI utilities bundled in the same bootc image
- `/charly-selkies:desktop-fonts` — Sibling font configuration for desktop containers

## Related Commands
- `/charly-build:build` — Build the bootc image including these desktop packages
- `/charly-vm:vm` — Run the bootc image as a VM to test the desktop apps

## When to Use This Skill

Use when the user asks about:

- Desktop applications in containers
- Chromium, VLC, KeePassXC, or cockpit
- The `desktop-apps` layer or its packages

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
