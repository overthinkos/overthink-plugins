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
| Install files | `tasks:` |

## Usage

```yaml
# image.yml
my-image:
  layers:
    - copr-desktop
```

## Used In Images

- `/ov-foundation:bazzite-ai` (disabled)

## Related Layers
- `/ov-selkies:desktop-apps` — Sibling desktop GUI application bundle
- `/ov-foundation:os-config` — Sibling bootc system config layer
- `/ov-foundation:os-system-files` — Sibling bootc system files overlay

## Related Commands
- `/ov-build:build` — Builds the bootc image including these COPR packages
- `/ov-advanced:vm` — Run the bootc image as a VM to test desktop packages

## When to Use This Skill

Use when the user asks about:

- CoolerControl fan/cooling management
- Ghostty terminal emulator
- Nerd Fonts installation (from Terra repo)
- WinBoat COPR package
- COPR repository packages for desktop use

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
