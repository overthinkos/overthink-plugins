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
| Install files | `candy.yml` (packages only) |

## Packages

RPM: `avahi-devel`, `avahi-glib-devel`, `eigen3-devel`, `glslang-devel`, `gstreamer1-devel`, `json-devel`, `libnotify-devel`, `librealsense`, `libuvc`, `libXrandr-devel`, `opencv`, `opencv-video`, `openhmd`, `openvr-api`, `openxr-devel`, `systemd-devel`, `vulkan-devel`, `vulkan-loader-devel`

## Usage

```yaml
# box.yml
my-vr:
  layers:
    - vr-streaming
```

## Used In Images

- `/charly-distros:bazzite`

## Related Layers
- `/charly-distros:cuda` — GPU compute sibling for ML/streaming workloads
- `/charly-distros:nvidia` — NVIDIA runtime sibling
- `/charly-selkies:pipewire` — audio/media stack pairing

## Related Commands
- `/charly-build:build` — rebuild after layer changes
- `/charly-core:shell` — inspect installed VR libraries

## When to Use This Skill

Use when the user asks about:

- VR or AR development libraries
- OpenXR, OpenVR, or GStreamer
- VR streaming setup
- The `vr-streaming` layer

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
