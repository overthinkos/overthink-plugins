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
| Security | `devices: [/dev/uinput, /dev/uhid]`, `cap_add: [NET_ADMIN]`, `group_add: [keep-groups]`, `mounts: [/run/user/1000/podman/podman.sock:/var/run/docker.sock:rw, /dev/input:/dev/input:rw, tmpfs:/run/udev:rw,size=1m, tmpfs:/run/user/1000/wolf:rw,size=10m]` |
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

## Standalone Deployment (Podman Quadlet)

Wolf can be deployed standalone using the official image and a rootful Podman quadlet — no `ov` needed. This is the recommended approach for initial testing and matches Wolf's upstream documentation.

### Prerequisites

- Podman with rootful socket enabled: `sudo systemctl enable --now podman.socket`
- NVIDIA driver: `cat /sys/module/nvidia/version` (needed to build driver volume)
- Host directory `/tmp/wolf-sockets` created: `sudo mkdir -p /tmp/wolf-sockets`
- Kernel module `uinput` loaded: `lsmod | grep uinput`

### Host udev rules

Wolf creates virtual input devices with vendor `0xAB00`. Install rules to restrict access:

```bash
sudo tee /etc/udev/rules.d/85-wolf-virtual-inputs.rules << 'EOF'
SUBSYSTEM=="misc", KERNEL=="uinput", MODE="0660", GROUP="input", TAG+="uaccess"
SUBSYSTEM=="misc", KERNEL=="uhid", MODE="0660", GROUP="input", TAG+="uaccess"
SUBSYSTEM=="input", ATTRS{id/vendor}=="ab00", MODE="0660", GROUP="input", TAG+="uaccess"
EOF
sudo udevadm control --reload-rules && sudo udevadm trigger
```

On Bazzite, `/etc/` is writable (only `/usr` is immutable). These rules survive reboots.

### Quadlet file

**File:** `/etc/containers/systemd/wolf.container`

```ini
[Unit]
Description=Wolf Game Streaming Server (Games on Whales)
Requires=network-online.target podman.socket
After=network-online.target podman.socket

[Container]
Image=ghcr.io/games-on-whales/wolf:stable
ContainerName=wolf
Network=host
SecurityLabelDisable=true
PodmanArgs=--ipc=host --device-cgroup-rule "c 13:* rmw"
# GPU — explicit NVIDIA devices + driver volume (NOT CDI — see gotcha #8)
AddDevice=/dev/dri
AddDevice=/dev/uinput
AddDevice=/dev/uhid
AddDevice=/dev/nvidia0
AddDevice=/dev/nvidiactl
AddDevice=/dev/nvidia-modeset
AddDevice=/dev/nvidia-uvm
AddDevice=/dev/nvidia-uvm-tools
AddDevice=/dev/nvidia-caps/nvidia-cap1
AddDevice=/dev/nvidia-caps/nvidia-cap2
Volume=nvidia-driver-vol:/usr/nvidia
Volume=/etc/wolf:/etc/wolf:rw
Volume=/run/podman/podman.sock:/var/run/docker.sock:rw
Volume=/dev/:/dev/:rw
Volume=/run/udev:/run/udev:rw
# XDG_RUNTIME_DIR — host bind mount at SAME path inside and outside container.
# Wolf's Podman introspection fails, so it uses container path as host path.
# By making them identical, child container Wayland socket mounts work correctly.
Volume=/tmp/wolf-sockets:/tmp/wolf-sockets:rw
Environment=XDG_RUNTIME_DIR=/tmp/wolf-sockets
Environment=WOLF_LOG_LEVEL=INFO
Environment=WOLF_CFG_FILE=/etc/wolf/cfg/config.toml
Environment=WOLF_PRIVATE_KEY_FILE=/etc/wolf/cfg/key.pem
Environment=WOLF_PRIVATE_CERT_FILE=/etc/wolf/cfg/cert.pem
Environment=HOST_APPS_STATE_FOLDER=/etc/wolf/state
Environment=WOLF_STOP_CONTAINER_ON_EXIT=TRUE
Environment=WOLF_RENDER_NODE=/dev/dri/renderD129
Environment=NVIDIA_DRIVER_VOLUME_NAME=nvidia-driver-vol

[Service]
Restart=always
TimeoutStartSec=900
ExecStartPre=-/usr/bin/podman rm --force --time 2 WolfPulseAudio

[Install]
WantedBy=multi-user.target
```

**NVIDIA driver volume setup** (required before first start):
```bash
# Build nvidia driver volume from host libraries
sudo podman volume create nvidia-driver-vol
VOLPATH=$(sudo podman volume inspect nvidia-driver-vol --format '{{.Mountpoint}}')
sudo mkdir -p "$VOLPATH"/{lib/gbm,bin,share/vulkan/icd.d,share/egl/egl_external_platform.d,share/glvnd/egl_vendor.d}

# Copy all nvidia libraries (dereference symlinks)
for lib in /usr/lib64/libnvidia-*.so.* /usr/lib64/libcuda*.so.* /usr/lib64/libnvoptix*.so.* \
           /usr/lib64/libnvcuvid*.so.* /usr/lib64/libEGL_nvidia*.so.* /usr/lib64/libGLESv*_nvidia*.so.* \
           /usr/lib64/libGLX_nvidia*.so.* /usr/lib64/libvdpau_nvidia*.so.* /usr/lib64/libnvenc*.so.*; do
    [ -f "$lib" ] && sudo cp -L "$lib" "$VOLPATH/lib/"
done

# Create soname symlinks
for f in "$VOLPATH/lib"/*.so.*.*; do
    name="${f%%.so.*}"; sudo ln -sf "$(basename $f)" "${name}.so.1" 2>/dev/null
done

# GBM backend symlink (critical for compositor DMA buffers)
sudo ln -sf ../libnvidia-allocator.so.1 "$VOLPATH/lib/gbm/nvidia-drm_gbm.so"

# Config files (dereference symlinks)
sudo cp -L /etc/vulkan/icd.d/nvidia_icd.json "$VOLPATH/share/vulkan/icd.d/" 2>/dev/null
sudo cp -L /usr/share/egl/egl_external_platform.d/*nvidia*.json "$VOLPATH/share/egl/egl_external_platform.d/" 2>/dev/null
sudo cp -L /usr/share/glvnd/egl_vendor.d/10_nvidia.json "$VOLPATH/share/glvnd/egl_vendor.d/" 2>/dev/null
sudo cp -L /usr/bin/nvidia-smi "$VOLPATH/bin/" 2>/dev/null
```

### Deployment gotchas (learned from real deployment)

1. **Dual-GPU systems:** Wolf defaults to `/dev/dri/renderD128`. On systems where the NVIDIA GPU is `renderD129` (common with AMD iGPU + NVIDIA dGPU), set `WOLF_RENDER_NODE=/dev/dri/renderD129`. Check with `ls -la /dev/dri/by-path/`.

2. **XDG_RUNTIME_DIR must exist:** Wolf's API server binds a Unix socket at `$XDG_RUNTIME_DIR/wolf.sock`. The directory must exist before Wolf starts. Use `Tmpfs=/tmp/sockets:rw,size=10m` in the quadlet.

3. **Port conflicts with Sunshine:** Wolf and Sunshine share GameStream ports (47984, 47989, 48010, etc.). Only one can run at a time. Stop all ov Sunshine services before starting Wolf.

4. **Socket read/write:** Mount the Podman socket as `:rw` (not `:ro`). Wolf needs write access to create and manage child containers.

5. **PulseAudio cleanup:** `ExecStartPre=-/usr/bin/podman rm --force WolfPulseAudio` removes stale audio sidecars from crashed sessions. Without this, new sessions may fail to create audio sinks.

6. **SELinux:** `SecurityLabelDisable=true` is required. Wolf needs unrestricted device access for GPU encoding and input injection.

7. **Container self-inspection:** Wolf logs `Unable to find docker mount for path` because it can't determine its own container ID on Bazzite/Podman. This is cosmetic — streaming works regardless.

8. **CDI doesn't work for apps that need a virtual compositor.** CDI (`AddDevice=nvidia.com/gpu=all`) injects GPU devices but NOT the GBM backend libraries needed by gst-wayland-display's DMA buffer allocator. Apps like Test Ball (no compositor) work with CDI, but Wolf UI and any app with `start_virtual_compositor = true` requires the manual nvidia driver volume approach. The compositor panics with `Failed to create GsCUDABuf` / `Failed to create DMA buffer: No such file or directory` without it.

9. **NVIDIA driver volume must include GBM backend.** The `nvidia-driver-vol` volume must have `lib/gbm/nvidia-drm_gbm.so -> ../libnvidia-allocator.so.1` and the full `libnvidia-allocator.so.*` chain. Wolf's `30-nvidia.sh` init script copies the GBM backend from the volume to `/usr/lib/x86_64-linux-gnu/gbm/` — without it, the Smithay compositor can't allocate GPU buffers.

10. **Podman introspection failure — XDG_RUNTIME_DIR must use host bind mount.** Wolf translates container-internal paths to host paths via self-introspection (`/proc/self/mountinfo` regex + Docker API). This fails on Podman because the regex is Docker-specific. When introspection fails, Wolf uses the container-internal path as the host path for child container bind mounts. **Fix:** Use a host bind mount with the SAME path inside and outside the container (`Volume=/tmp/wolf-sockets:/tmp/wolf-sockets:rw`). Do NOT use `Tmpfs` — with Tmpfs, Podman creates directories instead of mounting socket files into child containers, causing `Can't connect to a Wayland display` errors.

11. **Host socket directory must exist before start.** `sudo mkdir -p /tmp/wolf-sockets` — this directory persists across Wolf restarts. On Bazzite, `/tmp` is a tmpfs cleared on reboot, so the directory must be recreated after reboot (add to a startup script or systemd tmpfiles.d).

### Lifecycle

```bash
sudo mkdir -p /etc/wolf/cfg /etc/wolf/state
sudo systemctl daemon-reload
sudo systemctl start wolf.service        # start
sudo systemctl status wolf.service       # status
sudo journalctl -u wolf.service -f       # logs
sudo systemctl stop wolf.service         # stop
sudo systemctl restart wolf.service      # restart
curl -s "http://localhost:47989/serverinfo?uniqueid=test&uuid=test"  # API check
```

### Comparison: ov vs standalone vs upstream

| Aspect | ov (`ov enable wolf`) | Standalone quadlet | Upstream Wolf quadlet |
|--------|------|------------|----------------|
| Location | `~/.config/containers/systemd/ov-wolf.container` | `/etc/containers/systemd/wolf.container` | `/etc/containers/systemd/wolf.container` |
| Rootless/Rootful | Rootless (user) | Rootful (system) | Rootful (system) |
| GPU | CDI (`nvidia.com/gpu=all`) | Manual driver volume (CDI breaks compositor) | Manual driver volume + `/dev/nvidia*` |
| Socket | `/run/user/1000/podman/podman.sock` | `/run/podman/podman.sock` | `/run/podman/podman.sock` |
| Image | Built from source (ov layer) | Official `wolf:stable` | Official `wolf:stable` |
| Entrypoint | supervisord → wolf-wrapper | Wolf native | Wolf native |

## Input Pipeline Deep Dive

Wolf's input handling is fundamentally different from all Sunshine variants — and is the reason Wolf works where every Sunshine variant in ov currently has broken input.

### Comparison across all streaming variants

| Aspect | Wolf | Sunshine (Sway) | Sunshine (X11) | Sunshine (Mutter) |
|--------|------|-----------------|----------------|-------------------|
| Input library | inputtino (C++) | Sunshine built-in | Sunshine built-in | Sunshine built-in |
| Device creation | uinput + uhid | uinput only | uinput only | uinput only |
| Vendor ID | `0xAB00` | `0xBEEF` | `0xBEEF` | `0xBEEF` |
| Compositor discovery | Built-in fake-udev (C++) | Python fake-udev (287 lines) | XTest (native X11) | uinput (native) |
| Required capabilities | `NET_ADMIN` | `NET_ADMIN` | None | None |
| Required mounts | `/dev/input:rw`, `/run/udev` tmpfs | `/dev/input:rw`, `/run/udev` tmpfs | `/dev/uinput` only | `/dev/uinput` only |
| Gamepad support | Xbox, PS5 DualSense (gyro/haptics via uhid), Nintendo | Xbox, PS (uinput) | Xbox, PS | Xbox, PS |
| Multi-session input | Yes (independent per session) | No (single) | No | No |
| Status in ov | **Working** | Capture OK, input broken | **Broken** | **Input broken** |

### Why Wolf's input works

Wolf IS the compositor (Smithay/gst-wayland-display). When a Moonlight client sends input, Wolf's inputtino library creates virtual uinput devices, and gst-wayland-display receives the events directly — no external compositor to discover devices through.

The input flow is self-contained per session:
1. inputtino creates virtual devices via `/dev/uinput` (vendor `0xAB00`)
2. Wolf runs `mknod` inside the app container to expose the device
3. Wolf's built-in fake-udev (C++) sends `NETLINK_KOBJECT_UEVENT` to notify the app's compositor
4. The app's compositor (Sway or Gamescope inside the child container) picks up the device

All within Wolf's own session. No cross-service coordination, no timing races.

### Why every Sunshine variant has broken input

**Sunshine/Sway:** Container sysfs doesn't reflect host-created uinput devices. A 287-line Python `fake-udev` script sends synthetic NETLINK_KOBJECT_UEVENT messages to trick sway's libinput into discovering Sunshine's passthrough devices (vendor `0xBEEF`). This is fragile, timing-dependent, and sometimes fails silently.

**Sunshine/X11:** XTest is native X11 input injection — no udev or netlink needed. Despite this cleaner path, the X11 variant is also broken. No working Sunshine input path exists.

**Sunshine/Mutter:** GNOME's Mutter was expected to handle uinput natively (no fake-udev needed), but mouse/keyboard input is broken in practice. Capture works via XDG portal, but input fails.

### DualSense support (uhid)

Wolf uses `/dev/uhid` for PS5 DualSense controllers — adaptive triggers, gyroscope, accelerometer, and haptic feedback. No Sunshine variant supports this. The `uhid` path enables full controller passthrough at the HID protocol level, not just button/axis mapping.

### Host udev rules — two vendor IDs

Wolf and Sunshine use different vendor IDs for their virtual input devices:
- **Wolf:** `0xAB00` (inputtino)
- **Sunshine:** `0xBEEF` (Sunshine passthrough)

Systems running both need separate udev rules. The cgroup rule `c 13:* rmw` (major 13 = uinput) allows Wolf to create character devices inside spawned child containers — without it, virtual input in child containers fails silently.

## Audio Pipeline Deep Dive

Wolf eliminates the PipeWire dependency that all Sunshine variants require.

### Comparison

| Aspect | Wolf | Sunshine (ov) |
|--------|------|---------------|
| Audio server | Spawns PulseAudio container per session | PipeWire layer pre-installed |
| Host audio needed | No | PipeWire must be running |
| Multi-session audio | Independent sink per session | Shared PipeWire instance |
| Codec | Opus via GStreamer | Opus via Sunshine |
| Container deps | `pulseaudio-libs` (client only) | Full PipeWire + WirePlumber |
| Encryption | AES-CBC 128-bit | AES-CBC 128-bit |
| Failure mode | PulseAudio container spawn fails | PipeWire crash = no audio |
| Cleanup | `ExecStartPre=-podman rm WolfPulseAudio` | PipeWire restarts via supervisord |

### Self-contained audio

Wolf spawns a PulseAudio container (`ghcr.io/games-on-whales/pulseaudio:master`) per session via the Podman socket. No host audio server needed. This is critical on headless systems (like Bazzite) where PipeWire may be inactive.

On first start, Wolf downloads the PulseAudio image automatically. Subsequent starts reuse the cached image.

### Per-session isolation

Each Moonlight client gets its own virtual PulseAudio sink. Audio from one session never leaks into another. The GStreamer pipeline captures audio from the virtual sink → Opus encoder → `rtpmoonlightpay_audio` (RTP packetize, AES-CBC encrypt, FEC) → UDP stream.

### PulseAudio cleanup

The `ExecStartPre=-podman rm --force WolfPulseAudio` in the quadlet removes stale PulseAudio containers from crashed sessions. Without this, new sessions may fail to create audio sinks because the container name is already taken.

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

For standalone (non-ov) deployments, use Moonlight directly:
1. Add host IP in Moonlight client
2. Get PIN from logs: `sudo journalctl -u wolf.service | grep -i pin`
3. Enter PIN in Moonlight → apps appear

Wolf uses standard bridge networking with port mappings, so `ov moon` resolves ports normally.

## Related Layers

- `/ov-layers:sunshine` — Alternative streaming server (all variants currently have broken input)
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
- Standalone Wolf deployment without ov
- Input handling / fake-udev comparison
- Audio pipeline differences between Wolf and Sunshine
