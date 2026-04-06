---
name: merge
description: |
  Post-build layer optimization via merging consecutive small layers.
  MUST be invoked before any work involving: ov merge command, image layer reduction, merge configuration, or post-build optimization.
---

# ov merge -- Post-Build Layer Optimization

## Overview

Reduces image layer count by merging consecutive small layers. Uses `go-containerregistry` to load the image, groups consecutive layers below a size threshold, deduplicates filesystem entries (last writer wins), and reconstructs the image. Idempotent -- safe to run multiple times.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Merge single image | `ov merge <image>` | Merge layers in specified image |
| Dry run | `ov merge <image> --dry-run` | Show what would be merged without changing anything |
| Custom threshold | `ov merge <image> --max-mb N` | Set max layer size for merge candidates (default: 128 MB) |
| Merge all auto | `ov merge --all` | Merge all images that have `merge.auto: true` |

## Usage

### Basic Merge

```bash
# Merge consecutive small layers in an image
ov merge sway-browser-vnc

# Preview without changing
ov merge sway-browser-vnc --dry-run
```

### Custom Size Threshold

```bash
# Only merge layers smaller than 64 MB
ov merge sway-browser-vnc --max-mb 64
```

### Automatic Merge for All Configured Images

```bash
# Merge all images that opt in via images.yml
ov merge --all
```

## Configuration in images.yml

```yaml
images:
  sway-browser-vnc:
    merge:
      auto: true      # Include in `ov merge --all`
      max_mb: 128     # Size threshold (default: 128)
```

## How It Works

1. Load image via `go-containerregistry`
2. Group consecutive layers where each is below `max_mb`
3. Deduplicate filesystem entries across merged layers (last writer wins) and suppress whiteout conflicts
4. Reconstruct image with merged layers
5. Inline merge also runs automatically after each build level during `ov build`

## Whiteout Handling

OCI/Docker images use special "whiteout" files to represent file deletions across layers. When merging layers, these must be handled correctly to prevent `EEXIST` errors during overlay unpack.

**Three cases:**

1. **Regular whiteout** — A file `.wh.<name>` in layer N indicates that `<name>` was deleted. If an earlier layer contains the original file, the merge suppresses the original (keeps the whiteout marker so the deletion is preserved in the merged output).

2. **Opaque whiteout** — A file `.wh..wh..opq` in directory D means "the entire directory was replaced." All entries under D from earlier layers are suppressed. Only entries from the layer containing the opaque marker (and later layers) survive.

3. **Reintroduction supersedes whiteout** — If a file is deleted (whiteout in layer M) then re-created (same path in layer N, where N > M), the whiteout is suppressed and the re-introduced file is kept. This prevents the merged layer from containing both a file and its own whiteout, which would cause overlay unpack failures.

**Why this matters:** Without whiteout suppression, merged layers could contain contradictory entries (a file and its `.wh.*` marker coexisting), causing `EEXIST` errors when the container runtime unpacks the layer onto an overlay filesystem.

## Cross-References

- `/ov:build` -- building images (merge runs as part of build)
- `/ov:generate` -- Containerfile generation
