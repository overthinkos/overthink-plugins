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

- `/overthink-images:bazzite-ai` (disabled)

## When to Use This Skill

Use when the user asks about:

- CoolerControl fan/cooling management
- Ghostty terminal emulator
- Nerd Fonts installation (from Terra repo)
- WinBoat COPR package
- COPR repository packages for desktop use
