---
name: niri-desktop-sunshine
description: |
  GPU-accelerated Niri desktop with Sunshine game streaming.
  Use when working with Niri-based Sunshine streaming containers.
---

# niri-desktop-sunshine -- Niri desktop with Sunshine streaming

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `niri-desktop`, `sunshine-niri` |
| Install files | none (pure composition) |

## Usage

```yaml
# images.yml
sunshine-desktop-niri:
  base: nvidia
  layers:
    - niri-desktop-sunshine
  ports:
    - "9222:9222"
    - "47984:47984"
    - "47989:47989"
    - "47990:47990"
    - "48010:48010"
    - "47998:47998/udp"
    - "47999:47999/udp"
    - "48000:48000/udp"
```

## Comparison with sway-desktop-sunshine

| Component | sway-desktop-sunshine | niri-desktop-sunshine |
|-----------|----------------------|----------------------|
| Compositor | Sway (wlroots) | Niri (Smithay, built from source) |
| Headless mode | `WLR_BACKENDS=headless` | `NIRI_BACKEND=headless` (virtual outputs) |
| Capture | wlr-screencopy (working) | wlr-screencopy (pending — protocol gap) |
| Wrapper complexity | sway-wrapper (77 lines) | niri-wrapper (20 lines) |
| GPU workarounds | 4 env vars + --unsupported-gpu | None |

## Known Limitation

Sunshine capture does not work with niri headless mode because niri does not implement the `wlr-screencopy` protocol. See `/ov-layers:niri` for details and workaround timeline.

## Used In Images

- `/ov-images:sunshine-desktop-niri`

## Related Layers

- `/ov-layers:sway-desktop-sunshine` -- Sway variant (working capture)
- `/ov-layers:niri-desktop` -- base desktop (composed)
- `/ov-layers:sunshine-niri` -- Sunshine for Niri (included)
- `/ov:sun` -- Sunshine management commands
- `/ov:moon` -- Moonlight client protocol

## When to Use This Skill

Use when working with Niri-based Sunshine streaming desktop containers or comparing Niri vs Sway desktop stacks.
