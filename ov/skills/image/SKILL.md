---
name: image
description: |
  MUST be invoked before any work involving: image definitions in images.yml, image inheritance, defaults, platforms, builder configuration, or the image dependency graph.
---

# Image - Image Composition

## Overview

An **image** is a named build target in `images.yml`. Images compose layers into container images with configurable defaults, inheritance chains, platform targets, and builder configurations. The `ov` CLI resolves dependencies, generates Containerfiles, and builds images in the correct order.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| List images | `ov list images` | Images from images.yml |
| List build targets | `ov list targets` | Build targets in dependency order (includes auto-intermediates) |
| Inspect image | `ov inspect <image>` | Print resolved config as JSON |
| Inspect field | `ov inspect <image> --format <field>` | Print single field (tag, base, layers, ports, etc.) |
| Validate | `ov validate` | Check images.yml + layers |

## images.yml Structure

```yaml
defaults:
  registry: ghcr.io/overthinkos
  tag: auto                    # CalVer: YYYY.DDD.HHMM
  platforms:
    - linux/amd64
    - linux/arm64
  pkg: rpm
  builders:                    # build type → builder image
    pixi: fedora-builder
    npm: fedora-builder
    cargo: fedora-builder
  merge:
    auto: false
    max_mb: 128

images:
  fedora:
    base: "quay.io/fedora/fedora:43"
    pkg: rpm

  fedora-builder:
    base: fedora
    builds: [pixi, npm, cargo]  # declares what this builder can build
    layers:
      - pixi
      - nodejs
      - build-toolchain

  my-app:
    base: fedora
    layers:
      - supervisord
      - traefik
      - my-service
    ports:
      - "8080:8080"
    env:
      - MY_VAR=value
    env_file: "~/.config/my-app/.env"
    security:
      cap_add: [SYS_PTRACE]
```

## Inheritance Chain

Every setting resolves through: **image -> defaults -> hardcoded fallback** (first non-null wins).

| Field | Default | Description |
|-------|---------|-------------|
| `enabled` | `true` | Set `false` to disable (skipped by generate, validate, list) |
| `version` | `""` | CalVer version (`YYYY.DDD.HHMM`) of this image definition. Set manually |
| `status` | `""` (= `testing`) | `working`, `testing`, or `broken`. Effective status = worst of image + all layers |
| `info` | `""` | Free-form description. Aggregated with layer-level info in OCI labels |
| `base` | `quay.io/fedora/fedora:43` | External OCI image or name of another image |
| `bootc` | `false` | Adds `bootc container lint`, enables disk image builds |
| `platforms` | `["linux/amd64", "linux/arm64"]` | Target architectures |
| `tag` | `"auto"` | Image tag. `"auto"` for CalVer |
| `registry` | `""` | Container registry prefix |
| `pkg` | `"rpm"` | Tags: string `"rpm"` or list `["rpm", "fedora", "fedora:43"]`. First entry must be a format (rpm/deb/pac/aur). Tags control layer.yml sections and root.yml/user.yml tasks. Inherited from base image |
| `layers` | (required) | Layer list (image-specific, not inherited) |
| `ports` | `[]` | Runtime port mappings (`"host:container"` or `"port"`) |
| `user` | `"user"` | Username for non-root operations |
| `uid` | `1000` | User ID |
| `gid` | `1000` | Group ID |
| `merge` | `null` | Layer merge settings |
| `aliases` | `[]` | Command aliases |
| `builders` | `{}` | Build type → builder image map (inherited from base image + defaults) |
| `builds` | `[]` | What this builder image can build: `pixi`, `npm`, `cargo`, `aur` (not inherited) |
| `env` | `[]` | Runtime env vars (`KEY=VALUE`). Not inherited from defaults |
| `env_file` | `""` | Path to `.env` file for runtime injection. Not inherited |
| `security` | `null` | Container security options. Overrides layer-level security |
| `bind_mounts` | `[]` | Bind mount declarations (image-level only, not inherited) |
| `network` | `string` | Container network mode (default: shared `ov` network; set `host` for host networking) |
| `vm` | `VmConfig` | VM settings: `disk_size`, `root_size`, `ram`, `cpus`, `rootfs`, `kernel_args`, `ssh_port`, `transport`, `firmware`, `network` |
| `libvirt` | `[]string` | Raw libvirt XML snippets for VM domain configuration |

## Builders and Builds

Builder images provide build tools (pixi, npm, cargo, yay) for multi-stage builds without bloating final images. Three fields control this:

- **`builders:`** on images — map of build type → builder image name. Inherited: image → base image → defaults → `{}`.
- **`builds:`** on builder images — list declaring what the builder can build. Not inherited.
- **`pkg:`** — system package formats (`rpm`, `deb`, `pac`, `aur`). Inherited from base image.

```yaml
defaults:
  builders:
    pixi: fedora-builder
    npm: fedora-builder
    cargo: fedora-builder

images:
  fedora-builder:
    base: fedora
    builds: [pixi, npm, cargo]
    layers: [pixi, nodejs, build-toolchain]

  archlinux:
    base: "docker.io/library/archlinux:latest"
    pkg: pac
    builders:
      pixi: archlinux-builder
      npm: archlinux-builder
      cargo: archlinux-builder
      aur: archlinux-builder

  archlinux-builder:
    base: archlinux              # inherits pkg: pac AND builders: from archlinux
    builds: [pixi, npm, cargo, aur]
    layers: [pixi, nodejs, build-toolchain, yay]

  arch-test:
    base: archlinux              # inherits builders: from archlinux
    pkg: [pac, aur]              # override to add aur format
    layers: [arch-pac-test, arch-aur-test]
```

Each build type resolves its builder independently: **`image.builders[type]` → `base_image.builders[type]` → `defaults.builders[type]` → `""`**. This means you can use `fedora-builder` for pixi but `archlinux-builder` for npm on the same image.

Self-reference protection: after merging defaults/base, any `builders` entry pointing to the image itself is filtered out. Builder images can't use themselves as builders.

Validation checks that every builder referenced in `builders:` declares the matching capability in `builds:`.

Source: `ov/generate.go` (`builderRefForFormat`), `ov/graph.go` (`ResolveImageOrder`, `ImageNeedsBuilder`), `ov/validate.go` (`validateBuilders`).

## Internal Base Images

When `base` references another image in `images.yml`, the generator resolves it to the full registry/tag and creates a build dependency. The referenced image must be built first.

```yaml
images:
  fedora:
    base: "quay.io/fedora/fedora:43"

  my-app:
    base: fedora        # References fedora image above
    layers: [my-layer]
```

## Intermediate Images

When multiple images share the same base and common layer prefixes, `ov` auto-generates intermediate images at branch points to maximize cache reuse.

```
fedora (external)
  -> fedora-supervisord (auto: pixi + python + supervisord)
     -> fedora-test (adds: traefik, testapi)
     -> openclaw (adds: nodejs, openclaw)
```

Auto-intermediates are marked with `Auto: true` and appear in `ov list targets`.

### Algorithm

`ComputeIntermediates()` runs during generation:
1. `GlobalLayerOrder()` computes a deterministic layer ordering across all images, prioritizing layers by popularity (how many images need them) for cache efficiency.
2. Images are grouped by their direct parent (base). For each sibling group with 2+ images, a **prefix trie** is built from their relative layer sequences.
3. The trie is walked to detect branch points (where sibling layer sequences diverge). At each branch, an auto-intermediate image is created.
4. Original images are rebased to the nearest intermediate, so shared layers are built once.

Source: `ov/intermediates.go` (`ComputeIntermediates`, `GlobalLayerOrder`, `walkTrieScoped`).

## Versioning

CalVer: `YYYY.DDD.HHMM` (year, day-of-year, UTC time). Computed once per `ov generate`.

| `tag` value | Generated tag(s) |
|-------------|-----------------|
| `"auto"` | `YYYY.DDD.HHMM` + `latest` |
| `"nightly"` | `nightly` only |
| `"1.2.3"` | `1.2.3` only |

Override: `ov generate --tag <value>`.

## Runtime Environment Variables

The `env` and `env_file` fields inject environment variables into containers at runtime (not build time):

```yaml
images:
  my-app:
    env:
      - DB_HOST=localhost
      - LOG_LEVEL=info
    env_file: "~/.config/my-app/.env"
```

These are the lowest priority in the env resolution chain. CLI flags (`-e`, `--env-file`) and workspace `.env` take precedence. See `/ov:run` for the full priority chain.

Source: `ov/envfile.go` (`ResolveEnvVars`).

## Security Configuration

Image-level `security:` overrides layer-level security settings:

```yaml
images:
  my-app:
    security:
      privileged: true
      cap_add: [SYS_ADMIN]
      devices: [/dev/fuse]
      security_opt: [label:disable]
```

Image `security.privileged` replaces the layer-derived value. `cap_add`, `devices`, `security_opt` are appended to layer-collected values (deduplicated). Applied as container run arguments at runtime (not build time).

Source: `ov/security.go` (`CollectSecurity`).

## VM Configuration

For bootc images, the `vm:` section configures VM disk builds and runtime:

```yaml
images:
  my-bootc-image:
    bootc: true
    base: "quay.io/fedora/fedora-bootc:43"
    layers: [sshd, qemu-guest-agent, bootc-config]
    vm:
      disk_size: "10 GiB"
      ram: "4G"
      cpus: 2
      rootfs: ext4
      ssh_port: 2222
    libvirt:
      - "<filesystem type='mount'>...</filesystem>"
```

Resolution chain: **image-level vm: -> defaults vm: -> ov config -> hardcoded defaults**. See `/ov:deploy` for full VM management documentation.

## Common Workflows

### Add a New Image

Add an entry to `images.yml` with `base` and `layers`, then build:

```bash
# Edit images.yml
# Then:
ov build my-new-image
```

### Layer Images (inheritance)

Set `base` to another image name:

```yaml
images:
  nvidia:
    base: fedora
    layers: [cuda]
    platforms: [linux/amd64]

  ml-workstation:
    base: nvidia
    layers: [python-ml, jupyter]
```

### Disable an Image

```yaml
images:
  experimental:
    enabled: false
    base: fedora
    layers: [experimental-layer]
```

## Cross-References

- `/ov:layer` -- Layer definitions that compose into images
- `/ov:build` -- Building the defined images
- `/ov:deploy` -- Deploying built images (quadlet, bootc)

## When to Use This Skill

**MUST be invoked** when the task involves image definitions in images.yml, image inheritance, defaults, platforms, builder configuration, or the image dependency graph. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Pre-build. Define images before building. See also `/ov:layer` (layer authoring), `/ov:build` (building).
