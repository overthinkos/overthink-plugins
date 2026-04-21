---
name: fedora-remote
description: |
  Fedora image using remote layer references from GitHub. Demonstrates
  the @github.com/org/repo/layers/name:version remote layer syntax.
  MUST be invoked before building, deploying, configuring, or troubleshooting the fedora-remote image.
---

# fedora-remote

Fedora image built with remote layer references — layers fetched from GitHub at build time.

## Image Properties

| Property | Value |
|----------|-------|
| Base | quay.io/fedora/fedora:43 |
| Layers | (remote refs) |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## Remote Layer References

```yaml
layers:
  - "@github.com/overthinkos/overthink/layers/dev-tools:main"
  - "@github.com/overthinkos/overthink/layers/build-toolchain:v2026.64.1937"
  - "@github.com/overthinkos/overthink/layers/nodejs"
```

- `dev-tools:main` — pinned to main branch
- `build-toolchain:v2026.64.1937` — pinned to specific version tag
- `nodejs` — default branch (no version specified)

## Quick Start

```bash
ov image build fedora-remote
ov shell fedora-remote
```

## Key Layers (remote)

- `/ov-layers:dev-tools` — bat, ripgrep, neovim, gh, direnv, etc.
- `/ov-layers:build-toolchain` — gcc, cmake, autoconf, ninja, git
- `/ov-layers:nodejs` — Node.js + npm

## Verification

After `ov image build`:
- `ov image list` — image appears in list
- `ov shell fedora-remote` — interactive shell works

## Related Images
- `/ov-images:fedora` — local-layer parent base image
- `/ov-images:fedora-builder` — local-layer alternative bundling the same tooling

## Related Commands
- `/ov:build` — fetch remote layer refs and build the image
- `/ov:validate` — validate remote layer references resolve

## When to Use This Skill

**MUST be invoked** when the task involves the fedora-remote image, remote layer references, or the `@github.com/...` layer syntax. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
