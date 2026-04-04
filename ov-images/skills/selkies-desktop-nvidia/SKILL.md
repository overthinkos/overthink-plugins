# Image: selkies-desktop-nvidia

GPU-accelerated variant of selkies-desktop using the NVIDIA base image with CUDA toolkit.

## Definition

```yaml
selkies-desktop-nvidia:
  base: nvidia
  layers:
    - agent-forwarding
    - selkies-desktop
  ports:
    - "3000:3000"
    - "9222:9222"
  platforms:
    - linux/amd64
```

## Base

`nvidia` — Fedora 43 + RPM Fusion + CUDA toolkit + NVIDIA drivers.

## Difference from selkies-desktop

Same layers, but the `nvidia` base includes the full CUDA toolkit which may resolve the NVENC encoder initialization failure seen with the `fedora-nonfree` base. Use this variant for GPU-accelerated H.264 encoding via NVENC.

## Recording

Desktop video recording included via `wl-record-pixelflux` (part of selkies-desktop metalayer). With NVIDIA GPU, pixelflux-record may use NVENC for H.264 encoding (zero CPU overhead).

## Status

Not yet tested. The `fedora-nonfree` variant works with CPU encoding. This variant needs verification for NVENC hardware encoding.
