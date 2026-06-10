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
| Install files | `charly.yml` (packages only) |

## Packages

RPM: `golang-bin` · PAC: `go` · DEB: `golang-go` — full cross-distro parity.

## Usage

```yaml
# charly.yml
my-image:
  layers:
    - golang
```

## Used In Images

- (none currently enabled)

## Related Skills

- `/charly-coder:language-runtimes` -- also includes `golang-bin` alongside other runtimes
- `/charly-coder:rust`, `/charly-coder:nodejs` — sibling language runtimes
- `/charly-coder:build-toolchain` — C/C++ toolchain often paired with Go builds
- `/charly-distros:github-runner` — consumes go for Actions workflows
- `/charly-image:layer` — layer authoring
- `/charly-internals:go` — charly CLI is itself built from Go — development conventions

## When to Use This Skill

Use when the user asks about:

- Go compiler in containers
- Go development environment
- The `golang` layer

## Related

- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
