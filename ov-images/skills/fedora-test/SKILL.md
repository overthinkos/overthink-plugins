---
name: fedora-test
description: |
  Test image with Traefik reverse proxy and testapi service.
  Currently disabled. Used for development testing.
  MUST be invoked before building or troubleshooting the fedora-test image.
---

# fedora-test

Development test image with Traefik reverse proxy and a FastAPI test service.

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

- `/ov-layers:traefik` — Reverse proxy
- `/ov-layers:testapi` — FastAPI test service

## Related Images
- `/ov-images:fedora` — parent base image
- `/ov-images:valkey-test` — sibling test image (also disabled)

## Related Commands
- `/ov:build` — build the fedora-test image
- `/ov:validate` — validate the test layer composition

## When to Use This Skill

**MUST be invoked** when the task involves the fedora-test image or Traefik routing validation.
