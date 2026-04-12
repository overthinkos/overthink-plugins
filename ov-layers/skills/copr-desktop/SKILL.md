---
name: copr-desktop
description: |
  COPR and external desktop packages: CoolerControl, Ghostty terminal, Nerd Fonts, WinBoat.
  Use when working with COPR repositories or these desktop applications in bootc images.
---

# copr-desktop -- COPR desktop packages

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `root.yml` |

## Usage

```yaml
# images.yml
my-image:
  layers:
    - copr-desktop
```

## Used In Images

- `/ov-images:bazzite-ai` (disabled)

## Related Layers
- `/ov-layers:desktop-apps` — Sibling desktop GUI application bundle
- `/ov-layers:os-config` — Sibling bootc system config layer
- `/ov-layers:os-system-files` — Sibling bootc system files overlay

## Related Commands
- `/ov:build` — Builds the bootc image including these COPR packages
- `/ov:vm` — Run the bootc image as a VM to test desktop packages

## When to Use This Skill

Use when the user asks about:

- CoolerControl fan/cooling management
- Ghostty terminal emulator
- Nerd Fonts installation (from Terra repo)
- WinBoat COPR package
- COPR repository packages for desktop use
