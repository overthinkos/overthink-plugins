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
# images.yml
my-desktop:
  layers:
    - desktop-apps
```

## Used In Images

- `bazzite-ai` (disabled)

## When to Use This Skill

Use when the user asks about:

- Desktop applications in containers
- Chromium, VLC, KeePassXC, or cockpit
- The `desktop-apps` layer or its packages
