---
name: docker-ce
description: |
  Docker CE engine with buildx and compose plugins from the official Docker repository.
  Use when working with Docker, container builds, or Docker Compose.
---

# docker-ce -- Docker CE engine and plugins

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `root.yml` |

## Packages

RPM (from `docker-ce-stable` repo): `containerd.io`, `docker-buildx-plugin`, `docker-ce`, `docker-ce-cli`, `docker-compose-plugin`

## Usage

```yaml
# images.yml
my-image:
  layers:
    - docker-ce
```

## Used In Images

- `bazzite-ai` (disabled)

## Related Layers
- `/ov-layers:container-nesting` — Alternative podman-based nested container support
- `/ov-layers:kubernetes` — Sibling kubectl/Helm commonly paired with docker-ce
- `/ov-layers:github-actions` — Sibling layer needing a container engine for act runs

## Related Commands
- `/ov:build` — Build the bootc image including docker-ce packages
- `/ov:vm` — Run the bootc image as a VM to test the docker engine

## When to Use This Skill

Use when the user asks about:

- Docker CE inside containers
- Docker buildx or compose plugins
- The `docker-ce` layer or Docker repository setup
