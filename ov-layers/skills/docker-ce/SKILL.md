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
| Install files | `layer.yml`, `tasks:` |

## Packages

RPM (from `docker-ce-stable` repo): `containerd.io`, `docker-buildx-plugin`, `docker-ce`, `docker-ce-cli`, `docker-compose-plugin`

## Cross-distro coverage

`rpm:` (Fedora — Docker's yum repo), `pac:` (Arch — `docker` metapackage from `extra`), `deb:` — via distro-version tag sections `debian:13:` and `ubuntu:24.04:` because the upstream apt repo URL differs per distro codename (`https://download.docker.com/linux/debian trixie` vs `.../ubuntu noble`). Each tag section declares its own `repos:` block with the correct URL + GPG key; packages (`docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`) are identical. See `/ov:layer` "distro-version tag sections".

## Usage

```yaml
# image.yml
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

## Related

- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
