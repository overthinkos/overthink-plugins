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
| Status | **disabled** (set `enabled: true` in box.yml) |

## Purpose

Used for testing the Valkey layer (Redis-compatible fork). Runs as a supervisord service on port 6379.

## Key Layers

- `/ov-infrastructure:valkey` — Valkey data store
- `/ov-infrastructure:supervisord` — Process manager

## Related Images
- `/ov-distros:fedora` — parent base image
- `/ov-distros:fedora-test` — sibling test image

## Related Commands
- `/ov-build:build` — build the valkey-test image
- `/ov-core:service` — manage the supervised valkey service

## When to Use This Skill

**MUST be invoked** when the task involves the valkey-test image or Valkey service testing.

## Related

- `/ov-image:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
