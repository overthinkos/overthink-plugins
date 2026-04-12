---
name: gh
description: |
  GitHub CLI and git.
  Use when working with the gh layer.
---

# gh -- GitHub CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |
| Depends | none |

## Packages

RPM: `gh`, `git`

## Usage

```yaml
# images.yml or layer.yml
layers:
  - gh
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:dev-tools` — alternative bundle that also ships `gh`
- `/ov-layers:openclaw-full` — metalayer that includes gh
- `/ov-layers:agent-forwarding` — pairs with gh for SSH/GPG agent access

## Related Commands
- `/ov:secrets` — provision GITHUB_TOKEN for `gh auth login`
- `/ov:shell` — run gh interactively inside a container
- `/ov:build` — installs gh during image build

## When to Use This Skill

Use when the user asks about:
- GitHub CLI in containers
- Git and gh setup
- The `gh` layer
