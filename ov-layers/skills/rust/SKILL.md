---
name: rust
description: |
  Rust compiler and Cargo package manager via system packages (RPM/DEB).
  Use when working with Rust development or Cargo builds.
---

# rust -- Rust compiler and Cargo

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |

PATH additions: `~/.cargo/bin`

## Packages

RPM: `rust`, `cargo`
DEB: `rustc`, `cargo`

## Usage

```yaml
# images.yml
my-image:
  layers:
    - rust
```

## Used In Images

- No enabled images use this layer directly
- Transitive dependency of `language-runtimes` (used in `bazzite-ai`, disabled)

## Related Layers

- `/ov-layers:language-runtimes` -- includes rust as a dependency

## When to Use This Skill

Use when the user asks about:

- Rust compiler in containers
- Cargo builds or Rust development
- The `rust` layer
