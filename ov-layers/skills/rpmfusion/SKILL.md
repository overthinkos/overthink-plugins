---
name: rpmfusion
description: |
  RPM Fusion free and nonfree repository configuration for Fedora.
  Use when working with RPM Fusion repos, multimedia codecs, or nonfree packages.
---

# rpmfusion -- RPM Fusion repository configuration

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `root.yml` |

## Usage

```yaml
# images.yml
fedora-nonfree:
  base: fedora
  layers:
    - rpmfusion
```

## Used In Images

- `/ov-images:fedora-nonfree` (base for immich and other images needing nonfree packages)

## Related Layers
- `/ov-layers:ffmpeg` -- nonfree multimedia codecs installed from RPM Fusion
- `/ov-layers:immich` -- consumes nonfree codecs via fedora-nonfree

## Related Commands
- `/ov:build` -- build the fedora-nonfree base image
- `/ov:image` -- inspect base/image inheritance

## When to Use This Skill

Use when the user asks about:

- RPM Fusion repository setup
- Nonfree or multimedia packages on Fedora
- The `fedora-nonfree` base image
- Enabling additional Fedora repositories
