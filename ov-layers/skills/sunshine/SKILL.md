---
name: sunshine
description: |
  Sunshine game streaming server (Moonlight-compatible) with NVENC GPU encoding.
  Use when working with Sunshine, game streaming, Moonlight, or GPU-accelerated remote desktop.
---

# sunshine -- Game streaming server for Moonlight clients

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord`, `sway`, `pipewire` |
| Ports | `tcp:47984`, `47989`, `47990`, `tcp:48010`, `udp:47998`, `udp:47999`, `udp:48000` |
| Service | `sunshine` (supervisord, priority 20) |
| Security | `devices: [/dev/uinput]` |
| Volume | `sunshine-config` -> `~/.config/sunshine` |
| Install files | `user.yml`, `sunshine-wrapper` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `SWAY_CAPTURE` | `sunshine` |

The `SWAY_CAPTURE=sunshine` env var signals `sway-wrapper` to use `gles2` renderer on NVIDIA headless instead of `pixman`. This enables the full GPU pipeline: gles2 compositing + NVENC encoding.

## Packages

- `sunshine` (RPM, from COPR `lizardbyte/beta`)

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 47984 | TCP | GameStream HTTPS API |
| 47989 | HTTP | GameStream HTTP API |
| 47990 | HTTP | Web UI (config/pairing) |
| 48010 | TCP | RTSP session setup |
| 47998 | UDP | Video stream |
| 47999 | UDP | Control channel |
| 48000 | UDP | Audio stream |

## Usage

```yaml
# images.yml -- typically included via sway-desktop-sunshine composition
my-desktop:
  base: nvidia
  layers:
    - sway-desktop-sunshine
  ports:
    - "47990:47990"
    - "47998:47998/udp"
    - "47999:47999/udp"
    - "48000:48000/udp"
```

## GPU Pipeline

With Sunshine, the entire NVIDIA headless pipeline is GPU-accelerated:

```
Sway (gles2/NVIDIA) -> Sunshine (wlr-screencopy) -> NVENC (H.264/HEVC/AV1) -> Moonlight client
```

Compare to wayvnc: sway forced to pixman (CPU), wayvnc shows gray/blank screens (upstream bug).

Sunshine captures via `wlr-screencopy-unstable-v1` — the same Wayland protocol that `grim` uses — which works correctly with the gles2 renderer on NVIDIA headless.

## Startup Timing

The `sunshine-wrapper` uses the same two-phase wait as `wayvnc-wrapper`:
1. Wait for Wayland display socket (`wayland-0`)
2. Wait for sway IPC socket (`sway-ipc.*.sock`) + 2s delay

A default `sunshine.conf` is generated on first start with `capture=wlroots` and `encoder=nvenc`.

## Client Access

```bash
# Set credentials (first-time setup)
ov sun passwd sway-browser-sunshine --generate

# Get Web UI URL
ov sun url sway-browser-sunshine

# Pair a Moonlight client (enter PIN from Moonlight)
ov sun pair sway-browser-sunshine 1234

# Check status
ov sun status sway-browser-sunshine
```

Or via Web UI at `https://<host>:47990`. See `/ov:sun` for full CLI reference.

## Differences from VNC (wayvnc)

| Aspect | Sunshine/Moonlight | wayvnc/VNC |
|--------|-------------------|------------|
| Video encoding | H.264/HEVC/AV1 (GPU) | Raw framebuffer (CPU) |
| NVIDIA headless | Full GPU pipeline (gles2) | Forced pixman (gray screens) |
| Audio | Built-in (Opus) | Separate (PipeWire forwarding) |
| Input | Keyboard + mouse + gamepad | Keyboard + mouse only |
| Client | Moonlight (all platforms) | Any VNC viewer |
| Automation | Not programmable | `ov vnc` commands |

Note: `ov cdp` and `ov wl` work with both Sunshine and VNC images for programmatic automation.

## Used In Images

Part of `/ov-layers:sway-desktop-sunshine` composition.

## Related Layers

- `/ov:sun` -- Sunshine management commands (credentials, pairing, config)
- `/ov-layers:supervisord` -- process manager dependency
- `/ov-layers:sway` -- Wayland compositor (provides display)
- `/ov-layers:pipewire` -- audio server (PulseAudio compat for Sunshine)
- `/ov-layers:sway-desktop-sunshine` -- composition that includes sunshine
- `/ov-layers:wayvnc` -- VNC alternative (used in sway-desktop)

## When to Use This Skill

Use when the user asks about:

- Sunshine game streaming server
- Moonlight client setup
- GPU-accelerated remote desktop
- NVENC encoding in containers
- Alternative to VNC for desktop access
- `SWAY_CAPTURE` environment variable
