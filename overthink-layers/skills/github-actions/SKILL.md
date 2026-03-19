---
name: github-actions
description: |
  GitHub Actions local runner (act-cli) and guestfs-tools via COPR.
  Use when working with GitHub Actions, local CI testing, or act.
---

# github-actions -- Act CLI for local GitHub Actions

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `root.yml` |

## Packages

RPM (from COPR `goncalossilva/act`): `act-cli`, `guestfs-tools`

## Usage

```yaml
# images.yml
my-ci:
  layers:
    - github-actions
```

## Used In Images

- `bazzite-ai` (disabled)

## When to Use This Skill

Use when the user asks about:

- Running GitHub Actions locally
- The act CLI tool
- The `github-actions` layer
