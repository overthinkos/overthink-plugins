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

RPM: `golang-bin`

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

- `/ov-layers:language-runtimes` -- also includes `golang-bin` alongside other runtimes
- `/ov-layers:rust`, `/ov-layers:nodejs` — sibling language runtimes
- `/ov-layers:build-toolchain` — C/C++ toolchain often paired with Go builds
- `/ov-layers:github-runner` — consumes go for Actions workflows
- `/ov:layer` — layer authoring
- `/ov-dev:go` — ov CLI is itself built from Go — development conventions

## When to Use This Skill

Use when the user asks about:

- Go compiler in containers
- Go development environment
- The `golang` layer
