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
# images.yml
my-vr:
  layers:
    - vr-streaming
```

## Used In Images

- `bazzite-ai` (disabled)

## When to Use This Skill

Use when the user asks about:

- VR or AR development libraries
- OpenXR, OpenVR, or GStreamer
- VR streaming setup
- The `vr-streaming` layer
