---
name: ov
description: |
  Overthink CLI (ov) binary installed into container/VM images for in-container use.
  Use when working with ov binary deployment inside containers or the ov-full composition.
---

# ov -- Overthink CLI binary

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `root.yml` |

## Usage

```yaml
# images.yml -- typically used via ov-full composition
my-image:
  layers:
    - ov-full
```

## Used In Images

Part of the `ov-full` composition layer. Used transitively in:

- `/overthink-images:githubrunner` (via ov-full)

## Related Layers

- `/overthink-layers:ov-full` -- composition that includes ov + virtualization + gocryptfs + socat

## When to Use This Skill

Use when the user asks about:

- Installing the ov binary inside containers
- The ov-full composition layer
- In-container ov CLI usage
