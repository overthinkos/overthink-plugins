---
name: vr-streaming
description: |
  OpenXR, OpenVR, and GStreamer libraries for VR streaming and development.
  Use when working with VR, AR, streaming, or spatial computing.
---

# vr-streaming -- VR/AR streaming and development libraries

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |

## Packages

RPM: `avahi-devel`, `avahi-glib-devel`, `eigen3-devel`, `glslang-devel`, `gstreamer1-devel`, `json-devel`, `libnotify-devel`, `librealsense`, `libuvc`, `libXrandr-devel`, `opencv`, `opencv-video`, `openhmd`, `openvr-api`, `openxr-devel`, `systemd-devel`, `vulkan-devel`, `vulkan-loader-devel`

## Usage

```yaml
# image.yml
my-vr:
  layers:
    - vr-streaming
```

## Used In Images

- `/ov-images:bazzite-ai` (disabled)

## Related Layers
- `/ov-layers:cuda` — GPU compute sibling for ML/streaming workloads
- `/ov-layers:nvidia` — NVIDIA runtime sibling
- `/ov-layers:pipewire` — audio/media stack pairing

## Related Commands
- `/ov:build` — rebuild after layer changes
- `/ov:shell` — inspect installed VR libraries

## When to Use This Skill

Use when the user asks about:

- VR or AR development libraries
- OpenXR, OpenVR, or GStreamer
- VR streaming setup
- The `vr-streaming` layer

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
