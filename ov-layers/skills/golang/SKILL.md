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
# images.yml
my-image:
  layers:
    - golang
```

## Used In Images

- `aurora` (disabled)

## Related Layers

- `/ov-layers:language-runtimes` -- also includes `golang-bin` alongside other runtimes

## When to Use This Skill

Use when the user asks about:

- Go compiler in containers
- Go development environment
- The `golang` layer
