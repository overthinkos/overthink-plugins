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
| Install files | `charly.yml` (packages only) |

PATH additions: `~/.cargo/bin`

## Packages

RPM: `rust`, `cargo`
DEB: `rustc`, `cargo`

## Usage

```yaml
# charly.yml
my-image:
  layers:
    - rust
```

## Used In Images

- No enabled images use this layer directly
- Transitive dependency of `language-runtimes`

## When to use the `rust` layer vs the `build-toolchain` `rust`+`cargo` packages

Two places now ship a Rust toolchain. Choose based on **where** you need it:

| Need | Use |
|------|-----|
| Rustc/cargo at **runtime** in the final container (developer shells, CI runners, on-the-fly cargo builds) | Add `/charly-coder:rust` to the runtime layers list of the image |
| Rustc/cargo at **build time only**, for compiling a cdylib that's copied into the final image (e.g., pixelflux_wayland) | Already provided by `/charly-distros:fedora-builder` via `/charly-coder:build-toolchain`'s system `rust`+`cargo` packages — no extra layer needed |

Builder-stage cargo (the build-toolchain path) keeps the runtime image small because the
toolchain stays in the multi-stage builder and never lands in the final layers.

## Related Layers

- `/charly-coder:language-runtimes` -- includes rust as a dependency
- `/charly-coder:build-toolchain` -- now also ships `rust`+`cargo` for builder-stage compilation

## When to Use This Skill

Use when the user asks about:

- Rust compiler in containers
- Cargo builds or Rust development
- The `rust` layer

## Author + Test References

- `/charly-image:layer` — layer authoring reference (tasks, vars, env_provide, tests block syntax)
- `/charly-eval:eval` — declarative testing framework for the `eval:` block
