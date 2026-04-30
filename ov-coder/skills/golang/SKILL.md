---
name: golang
description: |
  Go programming language compiler via RPM package.
  Use when working with Go development or Go builds.
---

# golang -- Go compiler

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |

## Packages

RPM: `golang-bin` · PAC: `go` · DEB: `golang-go` — full cross-distro parity.

## Usage

```yaml
# image.yml
my-image:
  layers:
    - golang
```

## Used In Images

- `aurora` (disabled)

## Related Skills

- `/ov-coder:language-runtimes` -- also includes `golang-bin` alongside other runtimes
- `/ov-coder:rust`, `/ov-coder:nodejs` — sibling language runtimes
- `/ov-coder:build-toolchain` — C/C++ toolchain often paired with Go builds
- `/ov-foundation:github-runner` — consumes go for Actions workflows
- `/ov-build:layer` — layer authoring
- `/ov-dev:go` — ov CLI is itself built from Go — development conventions

## When to Use This Skill

Use when the user asks about:

- Go compiler in containers
- Go development environment
- The `golang` layer

## Related

- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
