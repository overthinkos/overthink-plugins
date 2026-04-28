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
| Install files | `layer.yml`, `tasks:` |

## Packages

RPM (from COPR `goncalossilva/act`): `act-cli`, `guestfs-tools`

## Cross-distro coverage

`rpm:` (Fedora), `pac:` (Arch). `deb:` (Debian/Ubuntu) — `act` and `actionlint` are curl-downloaded GitHub release binaries (distro-agnostic already); `guestfs-tools` is dropped from deb (not in Debian main). The layer's `tasks:` are format-agnostic so the per-distro divergence is minimal.

## Usage

```yaml
# image.yml
my-ci:
  layers:
    - github-actions
```

## Used In Images

- `bazzite-ai` (disabled)

## Related Layers
- `/ov-layers:github-runner` — distinct self-hosted runner layer (not act)
- `/ov-layers:docker-ce` — common companion for act workflow execution
- `/ov-layers:dev-tools` — typically paired in CI images

## Related Commands
- `/ov:build` — installs act-cli from COPR during image build
- `/ov:shell` — run `act` interactively to test workflows locally

## When to Use This Skill

Use when the user asks about:

- Running GitHub Actions locally
- The act CLI tool
- The `github-actions` layer

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
