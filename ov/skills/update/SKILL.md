---
name: update
description: |
  Update image and restart service with data sync.
  MUST be invoked before any work involving: ov update command, pulling new image versions, data seeding, force-seed, or updating deployed services.
---

# ov update -- Update Image and Restart

## Overview

Pull or build a new image version, optionally sync data from data layers into bind-backed volumes, then restart the service. Data sync uses MERGE mode by default -- adds new files without overwriting existing user modifications.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Update | `ov update <image>` | Pull new image, seed data, restart |
| Skip data sync | `ov update <image> --no-seed` | Pull and restart without data sync |
| Force overwrite | `ov update <image> --force-seed` | Overwrite existing data (cp -a) |
| Build instead of pull | `ov update <image> --build` | Build image locally before restart |
| Specific tag | `ov update <image> --tag v2.0` | Update to a specific tag |
| Named instance | `ov update <image> -i INSTANCE` | Update a named instance |
| Cross-image data | `ov update <image> --data-from <other>` | Seed data from a different image |

## Data Sync Modes

| Flag | Mode | Behavior |
|------|------|----------|
| `--seed` (default) | MERGE | `cp -an` -- adds new files, preserves existing |
| `--no-seed` | SKIP | No data sync, just pull and restart |
| `--force-seed` | OVERWRITE | `cp -a` -- overwrites all existing data |

## Usage

### Standard Update

```bash
# Pull latest image, merge new data files, restart
ov update jupyter
```

### Build and Update

```bash
# Build locally then update
ov update jupyter --build
```

### Force Data Reset

```bash
# Overwrite all data with fresh image defaults
ov update jupyter --force-seed
```

### Cross-Image Data Source

```bash
# Use data layers from a different image
ov update jupyter --data-from jupyter-custom
```

## Behavior by Mode

### Quadlet Mode

1. Pull/build new image
2. Sync data from data layers into bind-backed volumes (if `--seed`)
3. `systemctl --user restart ov-<image>.service`
4. Update `deploy.yml` with new `data_source`

### Direct Mode

1. Pull/build new image
2. Sync data from data layers into bind-backed volumes (if `--seed`)
3. Print restart instructions (manual restart required)

## Cross-References

- `/ov:config` -- initial deployment setup
- `/ov:start` -- start a service
- `/ov:build` -- building images locally
- `/ov:status` -- check service status after update
