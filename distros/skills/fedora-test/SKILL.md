---
name: fedora-test
description: |
  Test image with Traefik reverse proxy and testapi service.
  Currently disabled. Used for development testing.
  MUST be invoked before building or troubleshooting the fedora-test image.
---

# fedora-test

Development test image with Traefik reverse proxy and a FastAPI test service.

Lives in the **`overthinkos/fedora`** repo (git submodule at **`image/fedora`**),
`enabled: false`. Its `fedora` base comes from the main repo's `base.yml`,
reached as `ov.fedora` via the submodule's `import:` of the main repo under the
`ov` namespace; the `agent-forwarding`/`traefik`/`testapi` layers are pulled
by github reference. Build from the submodule:
`ov -C image/fedora image build fedora-test --include-disabled`.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, traefik, testapi |
| Platforms | linux/amd64, linux/arm64 |
| Ports | 8000 (Traefik HTTP), 8080 (Traefik dashboard) |
| Status | **disabled** (set `enabled: true` in image.yml) |

## Purpose

Used for testing Traefik routing, service health checks, and the `testapi` layer. The testapi service runs on port 9090 and is routed via `testapi.localhost` through Traefik.

## Key Layers

- `/ov-infrastructure:traefik` — Reverse proxy
- `/ov-infrastructure:testapi` — FastAPI test service

## Related Images
- `/ov-distros:fedora` — parent base image
- `/ov-distros:valkey-test` — sibling test image (also disabled)

## Related Commands
- `/ov-build:build` — build the fedora-test image
- `/ov-build:validate` — validate the test layer composition

## When to Use This Skill

**MUST be invoked** when the task involves the fedora-test image or Traefik routing validation.

## Related

- `/ov-image:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
