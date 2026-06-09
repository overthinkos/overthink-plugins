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
| Install files | `candy.yml`, `task:` |

## Packages

RPM (from COPR `goncalossilva/act`): `act-cli`, `guestfs-tools`

## Cross-distro coverage

`rpm:` (Fedora), `pac:` (Arch). `deb:` (Debian/Ubuntu) — `act` and `actionlint` are curl-downloaded GitHub release binaries (distro-agnostic already); `guestfs-tools` is dropped from deb (not in Debian main). The layer's `task:` are format-agnostic so the per-distro divergence is minimal.

## Usage

```yaml
# box.yml
my-ci:
  layers:
    - github-actions
```

## Used In Images

- (none currently enabled)

## Related Layers
- `/charly-distros:github-runner` — distinct self-hosted runner layer (not act)
- `/charly-coder:docker-ce` — common companion for act workflow execution
- `/charly-coder:dev-tools` — typically paired in CI images

## Related Commands
- `/charly-build:build` — installs act-cli from COPR during image build
- `/charly-core:shell` — run `act` interactively to test workflows locally

## When to Use This Skill

Use when the user asks about:

- Running GitHub Actions locally
- The act CLI tool
- The `github-actions` layer

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
