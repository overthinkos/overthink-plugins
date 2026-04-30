---
name: fastfetch
description: |
  Fast system information tool (neofetch successor).
  Use when working with the fastfetch layer.
---

# fastfetch -- Fast system information tool

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |
| Depends | none |

## Packages

RPM: `fastfetch`

## What It Does

Displays system information (OS, kernel, CPU, GPU, memory, etc.) with ASCII art logo. Active successor to the archived neofetch project. Useful for verifying container environment and creating visual demos.

## Usage

```yaml
# image.yml or layer.yml
layers:
  - fastfetch
```

## Included In

- `sway-desktop` metalayer (and all sway-desktop-* variants)
- `selkies-desktop` metalayer (and all selkies-desktop-* variants)

## Used In Images

- `/ov-selkies:sway-browser-vnc` (via `sway-desktop` metalayer)
- `/ov-openclaw:openclaw-sway-browser` (via `sway-desktop` metalayer)
- `/ov-openclaw:openclaw-ollama-sway-browser` (via `sway-desktop` metalayer)
- `/ov-selkies:selkies-desktop` (via `selkies-desktop` metalayer)
- `/ov-selkies:selkies-desktop-nvidia` (via `selkies-desktop` metalayer)

## Related Layers
- `/ov-selkies:sway-desktop` — Metalayer that pulls fastfetch in
- `/ov-selkies:selkies-desktop` — Metalayer that pulls fastfetch in
- `/ov-coder:dev-tools` — Sibling that also packages fastfetch via dnf

## Related Commands
- `/ov-build:build` — Builds the fastfetch RPM into the image
- `/ov-core:shell` — Interactive shell to run `fastfetch` for environment verification

## When to Use This Skill

Use when the user asks about:
- System information display in containers
- fastfetch or neofetch in containers
- The `fastfetch` layer

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
