---
name: wolf
description: |
  Wolf game streaming server image (Games on Whales) on fedora-nonfree with
  host networking and container orchestration via Podman socket.
  MUST be invoked before building, deploying, configuring, or troubleshooting the wolf image.
---

# wolf

Container-native game streaming server built from source on Fedora. Uses Wolf's Smithay-based Wayland compositor and GStreamer encoding pipeline. Spawns per-app containers via Podman socket for multi-user streaming.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora-nonfree (Fedora + RPM Fusion) |
| Layers | wolf |
| Platforms | linux/amd64 |
| Ports | 47984 (TCP), 47989, 47999/udp, 48010 (TCP), 48100/udp, 48200/udp |
| Security | `cap_add: [SYS_ADMIN]`, `network: host` |
| Registry | ghcr.io/overthinkos |

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 47984 | GameStream HTTPS API | TCP |
| 47989 | GameStream HTTP API / Web | HTTP |
| 47999 | Control channel | UDP |
| 48010 | RTSP session setup | TCP |
| 48100 | Video stream | UDP |
| 48200 | Audio stream | UDP |

## Quick Start

```bash
ov build wolf
ov start wolf
# Wait for Wolf to initialize (~10s)

# Verify Wolf is running
curl -s "http://localhost:47989/serverinfo?uniqueid=test&uuid=test"

# Check logs
ov logs wolf

# Pair with Moonlight client
ov moon pair wolf --auto
```

## Runtime Details

Wolf is fundamentally different from Sunshine — it's an **orchestrator** that manages per-app containers:

- **Host networking** — Required for UDP game streaming. No port mapping.
- **Podman socket mount** — `/run/user/1000/podman/podman.sock` -> `/var/run/docker.sock`. Wolf spawns PulseAudio and app containers via Docker API.
- **SYS_ADMIN capability** — Required for Wolf to manage container namespaces via the socket. Set at image level, not layer level.
- **Self-contained** — Wolf brings its own compositor (gst-wayland-display), audio (GStreamer + spawned PulseAudio container), and input handling (inputtino). No sway/pipewire layers needed.
- **GPU auto-detection** — `wolf-wrapper` detects NVIDIA/AMD/Intel at startup and configures encoding.

## Differences from Sunshine Images

| Aspect | wolf | sway-browser-sunshine |
|--------|------|----------------------|
| Base | fedora-nonfree | nvidia |
| Compositor | Built-in (Smithay) | Sway (wlroots) |
| Audio | GStreamer + spawned PA container | PipeWire layer |
| Network | Host (required for UDP) | Bridge with port mapping |
| Desktop | No desktop — Wolf is headless | Full Sway desktop + Chrome |
| Multi-session | Yes | No |
| App model | Per-app containers | Single container |
| Input | Direct uinput/uhid | fake-udev → sway libinput |

## Key Layers

- `/ov-layers:wolf` — Wolf layer (build from source, ports, security, service)

## Related Images

- `/ov-images:sway-browser-sunshine` — Sunshine-based streaming (desktop model)
- `/ov-images:sway-browser-sunshine-steam` — Sunshine + Steam gaming
- `/ov-images:sway-browser-vnc-moonlight` — Moonlight client for testing Wolf

## When to Use This Skill

Use when the user asks about the wolf image, Wolf deployment, container-native game streaming, Games on Whales integration, or the orchestrator streaming model vs Sunshine's desktop model.
