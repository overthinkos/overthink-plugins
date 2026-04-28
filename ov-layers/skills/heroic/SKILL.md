---
name: heroic
description: |
  Heroic Games Launcher for Epic, GOG, and Amazon Prime Gaming with mangohud and gamemode.
  Use when working with Heroic, Epic Games, GOG, or non-Steam game launchers in containers.
---

# heroic -- Heroic Games Launcher

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `sway` |
| Volumes | `heroic-config` -> `~/.config/heroic`, `heroic-games` -> `~/Games/Heroic` |
| Install files | `tasks:` |

## Packages

- `mangohud` (RPM, Fedora repos) — FPS overlay for games
- `gamemode` (RPM, Fedora repos) — Feral GameMode performance optimizer
- `heroic` (RPM, downloaded from GitHub releases in tasks: — not in Fedora repos)

## Usage

```yaml
# image.yml — typically used with sway-desktop-vnc + steam
sway-browser-vnc-steam-heroic:
  layers:
    - sway-desktop-vnc
    - steam
    - heroic
```

## Installation

Heroic is an Electron app distributed as an RPM on GitHub releases. The `tasks:` downloads and installs it via `dnf install -y /tmp/heroic.rpm` (auto-resolves deps). Version pinned to v2.20.1.

COPR repos exist (`atim/heroic-games-launcher`) but builds are currently failing — GitHub RPM is more reliable.

## CLI Flags

| Flag | Purpose |
|------|---------|
| `heroic --fullscreen` | Controller-friendly full-screen UI |
| `heroic --no-gui` | Start minimized to system tray |
| `heroic://launch/<runner>/<app>` | Protocol handler — launch a specific game directly |

## Game Stores

Heroic manages three game stores:
- **Epic Games Store** (via Legendary backend)
- **GOG** (via gogdl backend)
- **Amazon Prime Gaming** (via Nile backend)

## Wine/Proton Management

Heroic manages its own Wine/Proton independently from Steam:
- Downloads Wine-GE, Proton-GE, Wine-Lutris, etc. to `~/.config/heroic/tools/`
- Also supports UMU (Universal Proton runner)
- Does NOT depend on Steam's Proton installations

## Data Directories

| Path | Content |
|------|---------|
| `~/.config/heroic/` | Config, game settings, Wine/Proton tools |
| `~/Games/Heroic/` | Game installs + Wine prefixes |

Both are persistent named volumes.

## Related Layers

- `/ov-layers:steam` — Steam client (sibling game launcher)
- `/ov-layers:sway` — Wayland compositor (dependency)
- `/ov-layers:sway-desktop-vnc` — Desktop composition with VNC

## Used In Images

Not used in any current image definition. Standalone gaming layer requiring a Sway desktop image.

## When to Use This Skill

Use when the user asks about:

- Heroic Games Launcher installation
- Epic Games Store, GOG, or Amazon Prime Gaming in containers
- Non-Steam game launchers
- mangohud or gamemode configuration
- Wine/Proton management outside of Steam

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
