---
name: wolf
description: |
  Wolf game streaming server image (Games on Whales) on fedora-nonfree with
  container orchestration via Podman socket and standard port mappings.
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

- **Standard port mappings** — All 6 ports (TCP + UDP) mapped like other overthink images.
- **Podman socket mount** — `/run/user/1000/podman/podman.sock` -> `/var/run/docker.sock`. Wolf spawns PulseAudio and app containers via Docker API. Required unconditionally by Wolf's `docker::init()`.
- **Self-contained** — Wolf brings its own compositor (gst-wayland-display), audio (GStreamer + spawned PulseAudio container), and input handling (inputtino). No sway/pipewire layers needed.
- **GPU auto-detection** — `wolf-wrapper` detects NVIDIA/AMD/Intel at startup and configures encoding.

## Differences from Sunshine Images

| Aspect | wolf | sway-browser-sunshine |
|--------|------|----------------------|
| Base | fedora-nonfree | nvidia |
| Compositor | Built-in (Smithay) | Sway (wlroots) |
| Audio | GStreamer + spawned PA container | PipeWire layer |
| Network | Bridge with port mapping | Bridge with port mapping |
| Desktop | No desktop — Wolf is headless | Full Sway desktop + Chrome |
| Multi-session | Yes | No |
| App model | Per-app containers | Single container |
| Input | Direct uinput/uhid | fake-udev → sway libinput |

## Standalone Deployment (Podman Quadlet)

Wolf can be deployed without ov using the official image and a rootful Podman quadlet. See `/ov-layers:wolf` for the full deployment guide, input pipeline deep dive, and audio pipeline deep dive.

### Quick start

```bash
sudo mkdir -p /etc/wolf/cfg /etc/wolf/state /tmp/wolf-sockets
# Build nvidia-driver-vol (see /ov-layers:wolf for full script)
# Write /etc/containers/systemd/wolf.container (see /ov-layers:wolf for full file)
sudo systemctl daemon-reload
sudo systemctl start wolf.service
curl -s "http://localhost:47989/serverinfo?uniqueid=test&uuid=test"
```

### Troubleshooting

| Problem | Check | Fix |
|---------|-------|-----|
| No GPU / software encoding | `sudo journalctl -u wolf.service \| grep -i encoder` | Set `WOLF_RENDER_NODE=/dev/dri/renderD129` (dual-GPU systems) |
| GsCUDABuf panic / DMA buffer error | `sudo journalctl -u wolf.service \| grep -i 'DMA\|GsCUDA'` | Use manual nvidia-driver-vol with GBM backend, NOT CDI |
| Wolf UI "Can't connect to Wayland" | `ls -la /tmp/wolf-sockets/wayland-1` (should be socket, not dir) | Use `Volume=/tmp/wolf-sockets:/tmp/wolf-sockets:rw` NOT `Tmpfs` |
| SIGSEGV on startup | `sudo journalctl -u wolf.service \| grep -i bind` | Stop conflicting Sunshine services (`systemctl --user stop ov-sunshine-*`) |
| Socket errors | `sudo systemctl status podman.socket` | `sudo systemctl enable --now podman.socket` |
| Input not working | `cat /etc/udev/rules.d/85-wolf-virtual-inputs.rules` | Install host udev rules for vendor `0xAB00` |
| No audio | `sudo podman ps \| grep WolfPulseAudio` | Check Podman socket is `:rw`, restart wolf.service |
| Can't connect | `ss -tlnp \| grep 47989` | Check firewall: `sudo firewall-cmd --list-ports` |
| "docker mount" errors | `sudo podman logs wolf \| grep mount` | Cosmetic on Bazzite/Podman — streaming works regardless |

### Why Wolf instead of Sunshine

Every Sunshine variant in ov currently has broken input:
- **Sway:** Capture works, input broken (fake-udev fragile)
- **X11:** Broken
- **Mutter:** Capture works, input broken
- **Niri:** Capture broken
- **KWin:** Disabled

Wolf is the only Moonlight-compatible streaming server with working input, audio, and video. Its self-contained architecture (built-in compositor, per-session PulseAudio, inputtino) avoids the compositor integration problems entirely.

## Key Layers

- `/ov-layers:wolf` — Wolf layer (build from source, ports, security, service, standalone deployment, input/audio deep dives)

## Related Images

- `/ov-images:sway-browser-sunshine` — Sunshine-based streaming (all variants have broken input)
- `/ov-images:sway-browser-sunshine-steam` — Sunshine + Steam gaming (broken input)
- `/ov-images:sway-browser-vnc-moonlight` — Moonlight client for testing Wolf

## When to Use This Skill

Use when the user asks about the wolf image, Wolf deployment, container-native game streaming, Games on Whales integration, standalone Wolf without ov, or the orchestrator streaming model vs Sunshine's desktop model.
