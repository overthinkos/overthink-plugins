---
name: language-runtimes
description: |
  Multi-language runtime layer with Go, PHP, .NET, nodejs-devel, and python3-devel.
  Use when working with polyglot development or multiple language runtimes.
---

# language-runtimes -- Go, PHP, .NET, and development headers

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `nodejs`, `rust`, `python` |
| Install files | `layer.yml` (packages only) |

## Packages

RPM: `dotnet-sdk-9.0`, `golang-bin`, `golang-bazil-fuse-devel`, `libicu`, `php`, `python3-devel`, `python3-ramalama`, `nodejs-devel`

## Usage

```yaml
# images.yml
my-polyglot:
  layers:
    - language-runtimes
```

## Used In Images

- `bazzite-ai` (disabled)

## Related Layers

- `/ov-layers:nodejs` -- Node.js dependency
- `/ov-layers:rust` -- Rust dependency
- `/ov-layers:python` -- Python dependency

## When to Use This Skill

Use when the user asks about:

- Multiple language runtimes in one image
- Go, PHP, .NET SDK inside containers
- The `language-runtimes` layer
