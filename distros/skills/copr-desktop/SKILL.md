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
| Install files | `task:` |

## Usage

```yaml
# image.yml
my-image:
  layers:
    - copr-desktop
```

## Used In Images

- `/ov-distros:bazzite`

## Related Layers
- `/ov-selkies:desktop-apps` — Sibling desktop GUI application bundle
- `/ov-distros:os-config` — Sibling bootc system config layer
- `/ov-distros:os-system-files` — Sibling bootc system files overlay

## Related Commands
- `/ov-build:build` — Builds the bootc image including these COPR packages
- `/ov-vm:vm` — Run the bootc image as a VM to test desktop packages

## When to Use This Skill

Use when the user asks about:

- CoolerControl fan/cooling management
- Ghostty terminal emulator
- Nerd Fonts installation (from Terra repo)
- WinBoat COPR package
- COPR repository packages for desktop use

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
