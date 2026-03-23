---
name: wolf
description: |
  Wolf game streaming server (Games on Whales) — Moonlight-compatible orchestrator with
  Smithay-based compositor, built from source on Fedora. Container-native alternative to Sunshine.
  Use when working with Wolf, Games on Whales, or container-native game streaming.
---

# wolf -- Container-native game streaming server (Games on Whales)

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord`, `rust`, `build-toolchain` |
| Ports | `tcp:47984`, `47989`, `udp:47999`, `tcp:48010`, `udp:48100`, `udp:48200` |
| Services | `wolf` (priority 20) |
| Security | `devices: [/dev/uinput, /dev/uhid]`, `cap_add: [NET_ADMIN]`, `group_add: [keep-groups]`, `mounts: [/run/user/1000/podman/podman.sock:/var/run/docker.sock:rw, /dev/input:/dev/input:rw, tmpfs:/run/udev:rw,size=1m]` |
| Volumes | `wolf-config` -> `~/.config/wolf`, `wolf-state` -> `/var/lib/wolf` |
| Install files | `root.yml` (build from source), `user.yml`, `wolf-wrapper` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `WOLF_LOG_LEVEL` | `INFO` |
| `WOLF_CFG_FILE` | `~/.config/wolf/config.toml` |
| `WOLF_PRIVATE_KEY_FILE` | `~/.config/wolf/key.pem` |
| `WOLF_PRIVATE_CERT_FILE` | `~/.config/wolf/cert.pem` |
| `XDG_RUNTIME_DIR` | `/run/user/1000/wolf` (dedicated tmpfs — NOT `/tmp`, which corrupts host permissions via sibling container mounts) |
| `GST_GL_API` | `gles2` |
| `GST_GL_WINDOW` | `surfaceless` |
| `GST_PLUGIN_PATH` | `/usr/local/lib64/gstreamer-1.0` |

## Packages

**Runtime** (RPM, from RPM Fusion): `gstreamer1`, `gstreamer1-plugins-base`, `gstreamer1-plugins-good`, `gstreamer1-plugins-bad-freeworld`, `gstreamer1-plugins-ugly`, `gstreamer1-vaapi`, `gstreamer1-plugin-libav`, `libinput`, `libevdev`, `pulseaudio-libs`, `libdrm`, `libwayland-server`, `libxkbcommon`, `mesa-libEGL`, `mesa-libgbm`, `pciutils-libs`, `libunwind`, `libatomic`

**Build deps** (cleaned up after build): `clang-devel`, `libstdc++-static`, `glibc-static`, `boost-devel`, `boost-static`, GStreamer/OpenSSL/Wayland/Mesa `-devel` packages

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 47984 | TCP | GameStream HTTPS API (pairing, serverinfo) |
| 47989 | HTTP | GameStream HTTP API |
| 47999 | UDP | Control channel |
| 48010 | TCP | RTSP session setup |
| 48100 | UDP | Video stream |
| 48200 | UDP | Audio stream |

## Build from Source

Wolf is built natively on Fedora via `root.yml` (not extracted from Docker image):

1. **gst-wayland-display** — Custom Rust GStreamer plugin (Smithay-based Wayland compositor). Built with `cargo-c` (`cargo cinstall`) from `github.com/games-on-whales/gst-wayland-display`.
2. **Wolf binary** — C++20 CMake project with static Boost linking. Cloned from `github.com/games-on-whales/wolf`, built with Ninja.
3. **fake-udev** — Synthetic udev event sender for virtual input devices. Part of the Wolf build.

Build deps are cleaned up at the end of `root.yml` to reduce image size.

## Architecture: Wolf vs Sunshine

Wolf and Sunshine are both Moonlight-compatible but architecturally different:

| Aspect | Wolf | Sunshine |
|--------|------|----------|
| Compositor | Built-in Smithay (gst-wayland-display) | Requires external Sway |
| Audio | GStreamer pipeline (spawns PulseAudio container) | Requires PipeWire layer |
| Input | inputtino + uinput/uhid directly | fake-udev + sway libinput |
| Container model | Orchestrator — spawns per-app containers via Docker/Podman socket | Single container with Sway desktop |
| Multi-user | Multiple simultaneous streaming sessions | Single session per desktop |
| GPU encoding | VAAPI/NVENC/QuickSync (auto-detect) | NVENC-focused |
| Dependencies | `supervisord` only (self-contained) | `supervisord` + `sway` + `pipewire` |
| Install | Build from source (C++ + Rust) | RPM from COPR |

**Key insight**: Wolf is self-contained — it brings its own compositor, audio, and input handling. It does NOT need sway, pipewire, or dbus layers.

## Docker/Podman Socket

Wolf spawns per-app containers (PulseAudio, game sessions) via the Docker API. The Podman socket is bind-mounted at `/var/run/docker.sock`:

- **Rootless Podman**: `/run/user/1000/podman/podman.sock` (default in layer)
- **Rootful Podman**: `/run/podman/podman.sock` (override via deploy.yml)
- **Docker**: `/var/run/docker.sock` (override via deploy.yml)

No special capabilities beyond `NET_ADMIN` (declared in `layer.yml` for input device injection) are needed for the socket mount.

## GPU Pipeline

```
Wolf (gst-wayland-display/Smithay) -> GStreamer (VAAPI/NVENC/x264) -> Moonlight client
```

Wolf's compositor directly exposes raw framebuffers to the GStreamer encoding pipeline — zero-copy on supported hardware. The `wolf-wrapper` detects GPU type at startup:
- NVIDIA: sets `WOLF_RENDER_NODE`, `GST_GL_API=gles2`
- AMD/Intel: auto-detects render node, uses VAAPI
- No GPU: software encoding fallback (x264/x265)

## Startup Sequence (wolf-wrapper)

1. GPU detection (NVIDIA via nvidia-smi → AMD/Intel via renderD* → software)
2. Wait for Docker/Podman socket (30s timeout, warns if missing)
3. Set up `/run/udev/data` and `/run/udev/control` for input handling
4. Ensure config directory exists and is writable
5. Generate default TOML config if none exists
6. `exec wolf`

## Configuration

Wolf uses TOML config at `~/.config/wolf/config.toml`. On first start, Wolf rewrites the initial config with a full default including app profiles. Key settings:

- `hostname` — Name shown in Moonlight clients
- `support_hevc` — HEVC codec support
- `paired_clients` — Authorized Moonlight devices
- `profiles` — App definitions with Docker runner configs

## Client Access

Wolf uses the same GameStream protocol as Sunshine. `ov moon` commands work:

```bash
# Check server status (no pairing needed)
ov moon status wolf

# Pair as Moonlight client
ov moon pair wolf --auto

# List available apps
ov moon apps wolf
```

**Note**: `ov moon` requires the container to have port mappings. Wolf uses `network: host`, so `ov moon` may need Go-side changes to resolve host-networked containers. Use `curl http://localhost:47989/serverinfo` to verify directly.

## Related Layers

- `/ov-layers:sunshine` — Alternative streaming server (desktop-centric, NVENC-focused)
- `/ov-layers:moonlight` — Moonlight client for testing
- `/ov-layers:supervisord` — Process manager dependency
- `/ov-layers:rust` — Rust compiler (build dependency)
- `/ov-layers:build-toolchain` — C/C++ build tools (build dependency)

## When to Use This Skill

Use when the user asks about:

- Wolf game streaming server
- Games on Whales
- Container-native game streaming (orchestrator model)
- gst-wayland-display / Smithay compositor
- Alternative to Sunshine
- Multi-user game streaming
- Building Wolf from source on Fedora
