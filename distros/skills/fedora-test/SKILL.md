---
name: fedora-test
description: |
  Test image with Traefik reverse proxy and testapi service.
  Currently disabled. Used for development testing.
  MUST be invoked before building or troubleshooting the fedora-test image.
---

# fedora-test

Development test image with Traefik reverse proxy and a FastAPI test service.

Lives in the **`overthinkos/fedora`** repo (git submodule at **`box/fedora`**).
Its `fedora` base is bare-local in the same self-contained submodule
(`import: []`) — `base: fedora`; the
`agent-forwarding`/`traefik`/`testapi` layers are pulled by github reference.
Build from the submodule: `charly -C box/fedora box build fedora-test`.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, traefik, testapi |
| Platforms | linux/amd64, linux/arm64 |
| Ports | 8000 (Traefik HTTP), 8080 (Traefik dashboard) |
| Status | enabled |

## Purpose

Used for testing Traefik routing, service health checks, and the `testapi` layer. The testapi service runs on port 9090 and is routed via `testapi.localhost` through Traefik.

## Key Layers

- `/charly-infrastructure:traefik` — Reverse proxy
- `/charly-infrastructure:testapi` — FastAPI test service

## Related Images
- `/charly-distros:fedora` — parent base image
- `/charly-distros:valkey-test` — sibling test image (disabled)

## Related Commands
- `/charly-build:build` — build the fedora-test image
- `/charly-build:validate` — validate the test layer composition

## When to Use This Skill

**MUST be invoked** when the task involves the fedora-test image or Traefik routing validation.

## Related

- `/charly-image:image` — image family umbrella (`box:` entries in `charly.yml`, build/validate/inspect/list)
