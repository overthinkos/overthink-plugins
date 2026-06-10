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
| Install files | `charly.yml` (packages only) |
| Depends | none |

## Packages

RPM: `fastfetch`

## What It Does

Displays system information (OS, kernel, CPU, GPU, memory, etc.) with ASCII art logo. Active successor to the archived neofetch project. Useful for verifying container environment and creating visual demos.

## Usage

```yaml
# box or candy charly.yml
candy:
  - fastfetch
```

## Included In

- `sway-desktop` metalayer (and all sway-desktop-* variants)
- `selkies-desktop` metalayer (and all selkies-desktop-* variants)

## Used In Images

- `/charly-selkies:sway-browser-vnc` (via `sway-desktop` metalayer)
- `/charly-selkies:selkies-labwc` (via `selkies-desktop` metalayer)
- `/charly-selkies:selkies-labwc-nvidia` (via `selkies-desktop` metalayer)

## Related Layers
- `/charly-selkies:sway-desktop` — Metalayer that pulls fastfetch in
- `/charly-selkies:selkies-desktop-layer` — Metalayer that pulls fastfetch in
- `/charly-coder:dev-tools` — Sibling that also packages fastfetch via dnf

## Related Commands
- `/charly-build:build` — Builds the fastfetch RPM into the image
- `/charly-core:shell` — Interactive shell to run `fastfetch` for environment verification

## When to Use This Skill

Use when the user asks about:
- System information display in containers
- fastfetch or neofetch in containers
- The `fastfetch` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
