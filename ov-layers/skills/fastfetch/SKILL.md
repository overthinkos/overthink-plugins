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
# images.yml or layer.yml
layers:
  - fastfetch
```

## Included In

- `sway-desktop` metalayer (and all sway-desktop-* variants)
- `selkies-desktop` metalayer (and all selkies-desktop-* variants)

## When to Use This Skill

Use when the user asks about:
- System information display in containers
- fastfetch or neofetch in containers
- The `fastfetch` layer
