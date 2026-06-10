---
name: valkey-test
description: |
  Test image with Valkey (Redis-compatible) key-value store.
  Currently disabled. Used for development testing.
  MUST be invoked before building or troubleshooting the valkey-test image.
---

# valkey-test

Development test image with Valkey in-memory data store.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, supervisord, valkey |
| Platforms | linux/amd64, linux/arm64 |
| Ports | 6379 (Valkey) |
| Status | **disabled** (set `enabled: true` in charly.yml) |

## Purpose

Used for testing the Valkey layer (Redis-compatible fork). Runs as a supervisord service on port 6379.

## Key Layers

- `/charly-infrastructure:valkey` — Valkey data store
- `/charly-infrastructure:supervisord` — Process manager

## Related Images
- `/charly-distros:fedora` — parent base image
- `/charly-distros:fedora-test` — sibling test image

## Related Commands
- `/charly-build:build` — build the valkey-test image
- `/charly-core:service` — manage the supervised valkey service

## When to Use This Skill

**MUST be invoked** when the task involves the valkey-test image or Valkey service testing.

## Related

- `/charly-image:image` — image family umbrella (`image:` entries in `charly.yml`, build/validate/inspect/list)
