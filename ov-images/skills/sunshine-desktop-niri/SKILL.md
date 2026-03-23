---
name: sunshine-desktop-niri
description: |
  GPU-accelerated Niri desktop with Sunshine streaming and Chrome browser.
  Use when working with Niri-based Sunshine streaming on NVIDIA.
---

# sunshine-desktop-niri

GPU-accelerated Niri desktop with Sunshine game streaming and Chrome browser. Built from QaidVoid/niri fork with virtual output support. Uses Smithay compositor instead of Sway/wlroots.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia (fedora + CUDA) |
| Layers | niri-desktop-sunshine |
| Platforms | linux/amd64 |
| Ports | 9222, 47984, 47989, 47990, 48010, 47998/udp, 47999/udp, 48000/udp |
| Registry | ghcr.io/overthinkos |

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 9222 | Chrome DevTools | HTTP |
| 47990 | Sunshine Web UI | HTTPS |
| 47998 | Video stream | UDP |
| 47999 | Control channel | UDP |
| 48000 | Audio stream | UDP |

## Quick Start

```bash
ov build sunshine-desktop-niri
ov start sunshine-desktop-niri
# Wait for services to start (~15s)
ov sun passwd sunshine-desktop-niri --generate   # Set Sunshine Web UI credentials
ov moon pair sunshine-desktop-niri --auto        # Pair as Moonlight client
ov sun status sunshine-desktop-niri              # Verify health
# Sunshine Web UI at https://localhost:47990
# Chrome DevTools at localhost:9222
```

## Known Limitation

Sunshine screen capture does not work with niri headless mode because niri (Smithay) does not implement the `wlr-screencopy` Wayland protocol. All other services (compositor, input, audio, Sunshine web UI) work correctly. Pending upstream Sunshine support for `ext-image-copy-capture-v1`.

For working Sunshine streaming today, use `/ov-images:sway-browser-sunshine` instead.

## Architecture: Niri vs Sway

| Aspect | sunshine-desktop (Niri) | sway-browser-sunshine (Sway) |
|--------|------------------------|------------------------------|
| Compositor | Niri (Smithay, Rust) | Sway (wlroots, C) |
| Built from | Source (QaidVoid/niri fork) | RPM package |
| Headless | `NIRI_BACKEND=headless` | `WLR_BACKENDS=headless` |
| Virtual outputs | HEADLESS-1 (1920x1080) | HEADLESS-1 (1920x1080) |
| Capture status | Not working (protocol gap) | Working (wlr-screencopy) |
| Wrapper complexity | 20 lines | 77 lines |
| GPU workarounds | None | 4 env vars + --unsupported-gpu |

## Key Layers

- `/ov-layers:niri-desktop-sunshine` -- full desktop composition with Sunshine
- `/ov-layers:niri` -- Smithay compositor (built from source)
- `/ov-layers:cuda` -- GPU support (via nvidia base)

## Related Images

- `/ov-images:sway-browser-sunshine` -- Sway variant (working capture)
- `/ov-images:sway-browser-sunshine-steam` -- Sway + Steam gaming
- `/ov-images:wolf` -- Container-native streaming (different architecture)

## When to Use This Skill

Use when the user asks about the sunshine-desktop image, Niri-based Sunshine streaming, or comparing Niri vs Sway desktop architectures.
