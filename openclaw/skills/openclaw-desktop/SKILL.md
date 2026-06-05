---
name: openclaw-desktop
description: >
  Use when working with the openclaw-desktop image — the all-in-one CachyOS
  power image that fuses the selkies streaming desktop, the openclaw-full
  gateway + AI CLIs (claude-code/codex/gemini), a CPU ollama, and the full
  nested ov toolchain (build images / run nested rootless pods / launch
  rootless libvirt VMs from a terminal inside the browser-accessible desktop).
---

# openclaw-desktop

The all-in-one workstation image: a browser-accessible Wayland **streaming
desktop** that also runs the **OpenClaw gateway + its full tool/skill stack**,
a **CPU Ollama** inference server, and the **complete `ov` toolchain** — all as
uid 1000 `user` with no `--privileged` and no added capabilities. Open
`https://localhost:3000`, get a labwc desktop with Chrome, and from a terminal
inside it you can `ov box build`, run nested rootless pods, launch rootless
libvirt VMs, drive the OpenClaw gateway on :18789, and hit a local Ollama on
:11434.

## Definition

```yaml
openclaw-desktop:
  base: cachyos.cachyos        # Arch-derived, pacman/AUR, x86_64_v3 (via the `cachyos` import namespace)
  build: [pac, aur]            # required — selkies' chrome (AUR google-chrome) + wl-tools (AUR wlrctl)
  candy:
    - agent-forwarding
    - selkies-desktop          # full streaming desktop stack
    - openclaw-full            # gateway + 27 tools incl. claude-code/codex/gemini
    - ollama                   # CPU ollama (GPU-agnostic layer)
    - ov                       # the full toolchain: ov binary + virtualization + gocryptfs + socat
    - container-nesting        # nested rootless podman/buildah/skopeo
    - golang
    - gh
    - dbus
    - ov
  port:
    - "3000:3000"              # Selkies web UI (Traefik HTTPS)
    - "9222:9222"              # Chrome DevTools Protocol (cdp-proxy)
    - "9224:9224"              # chrome-devtools-mcp (Streamable HTTP)
    - "2222:2222"              # sshd-wrapper
    - "18789:18789"            # OpenClaw gateway
    - "11434:11434"            # Ollama API
  platform: [linux/amd64]
```

No `uid:`/`gid:`/`user:`/`network:` override — inherits `1000/1000/user` and the
default `ov` bridge network. Rootless is the whole design.

## Base — CachyOS, CPU

`base: cachyos.cachyos` (`/ov-distros:cachyos`, reached via the `cachyos` import
namespace) is the Arch-derived `x86_64_v3` base. The
image is **CPU-only** — there is no `nvidia`/`cuda` layer. Ollama auto-detects
the absence of a GPU and runs CPU inference; selkies streams via the CPU x264
encoder. `build: [pac, aur]` is mandatory (not inherited): selkies' `chrome`
layer compiles `google-chrome` from AUR and `wl-tools` builds `wlrctl`;
inheriting cachyos's bare `[pac]` would gate out the AUR builder and silently
drop the browser. The `aur → arch-builder` builder map comes from the cachyos
base.

## Resolved security posture (OCI label)

| Field | Value |
|---|---|
| `cap_add` | **(empty)** |
| `security_opt` | `[unmask=/proc/*]` (from `/ov-distros:container-nesting`) |
| `devices` | `[/dev/fuse, /dev/net/tun]` (from `/ov-distros:container-nesting`) |
| `shm_size` | `1g` (from `/ov-selkies:chrome`) |
| `memory_max` | `6g` (from `/ov-selkies:chrome`) |
| `privileged` | `false` |
| Network | default ov bridge (NOT host) |
| UID / user | `1000 / user` |

**No `--privileged`. No `cap_add: ALL`. No `seccomp=unconfined`. No
`label=disable`.** The only security relaxation is `unmask=/proc/*` — surgical,
documented in `/ov-distros:container-nesting` with the full kernel-level RCA.

## The four fused stacks

| Stack | Layers | What it gives you |
|---|---|---|
| Streaming desktop | `selkies-desktop` (chrome, chrome-cdp, labwc, waybar, pipewire, swaync, pavucontrol, wl-tools, selkies, sshd, …) | labwc Wayland desktop streamed over HTTPS:3000; Chrome + CDP:9222 + chrome-devtools-mcp:9224; sshd:2222 |
| OpenClaw + tools | `openclaw-full` (openclaw gateway + claude-code, codex, gemini + 24 more tools) | AI gateway on :18789; `claude` / `codex` / `gemini` CLIs at `${HOME}/.npm-global/bin/`; playwright now drives the desktop's real Chrome (synergy) |
| LLM inference | `ollama` | CPU Ollama API on :11434; `ollama` host alias; `models` volume at `~/.ollama` |
| Nested ov toolchain | `ov` + `container-nesting` + `golang` + `gh` | `ov box build`, nested rootless podman/buildah/skopeo, rootless libvirt VMs, gocryptfs encrypted volumes, socat relays |

**Browser synergy:** `openclaw-full` ships `playwright` but deliberately omits a
system browser (it's normally headless). Fusing it with `selkies-desktop`'s
`chrome` + `chrome-cdp` stack gives playwright a real browser + CDP on :9222 —
a pure enhancement, no conflict.

## Ports

| Port | Service |
|---|---|
| 3000 | Selkies web UI (Traefik HTTPS, self-signed) |
| 9222 | Chrome DevTools Protocol (via `cdp-proxy`) |
| 9224 | `chrome-devtools-mcp` (Streamable HTTP) |
| 2222 | `sshd-wrapper` (supervisord-managed) |
| 18789 | OpenClaw gateway + Control UI |
| 11434 | Ollama API |

All six are distinct — no collisions. For multi-instance deployments alongside a
running selkies/openclaw instance holding these host ports, remap via
`ov config openclaw-desktop -p 3010:3000 -p 9232:9222 -p 9242:9224 -p 2232:2222 -p 18790:18789 -p 11444:11434`.

## Quick Start

```bash
ov box build openclaw-desktop
ov config openclaw-desktop
ov start openclaw-desktop
# Desktop:  https://localhost:3000 (accept the self-signed cert)
# Gateway:  http://localhost:18789
# Ollama:   curl http://localhost:11434/api/tags
ov shell openclaw-desktop -c "ollama pull llama3"
```

## Nested rootless podman — how it works here

See `/ov-distros:container-nesting` for the full kernel-level RCA. The short
version: podman's rootless outer container injects `linux.maskedPaths`; the
kernel's `mount_too_revealing()` then refuses any fresh `/proc` mount from the
inner container. `unmask=/proc/*` tells podman not to emit those masks on the
outer, so the inner `/proc` mount has nothing to mismatch with. Verified:

```bash
ov shell openclaw-desktop -c 'podman run --rm quay.io/libpod/alpine:latest /bin/true'
# zero extra caps, uid 1000
```

## Nested rootless VMs — how it works here

See `/ov-infrastructure:virtualization`. `virtqemud --timeout 0` +
`virtnetworkd --timeout 0` run as supervisord programs at uid 1000; libvirt
`qemu:///session` keys off `$XDG_RUNTIME_DIR` and needs no CAP_SYS_ADMIN — only
`/dev/kvm` passthrough.

```bash
ov shell openclaw-desktop -c 'virsh -c qemu:///session list --all'
```

### Two-level nested-virtualization (end-to-end `ov vm` run-through)

The full `ov vm build/create/ssh/stop/destroy` lifecycle completes inside the
rootless pod, two levels of KVM nesting deep:

1. Host (rootless podman, uid 1000) runs `ov-openclaw-desktop`.
2. Inside it, `ov vm build <bootc-image> --transport containers-storage`
   auto-falls back to `engine.rootful=machine` (no host `sudo` in reach), which
   spawns a podman-machine VM (nested VM #1) via KVM passthrough and runs
   `bootc install to-disk`.
3. `ov vm create <bootc-image> -i smoke --ssh-key generate` boots the produced
   qcow2 as a QEMU user-net VM (nested VM #2).
4. `ov vm ssh <bootc-image> -i smoke -- uname -a` returns the guest kernel.
5. `ov vm destroy --disk` cleans up.

No `--privileged`, no `cap_add`, no seccomp relaxations beyond the baked
`[unmask=/proc/*]`. Only `/dev/kvm` passthrough.

### Cross-storage loading for private bootc images

The podman-machine in step 2 has its own rootful storage, separate from the
outer container's nested rootless store. Load a private bootc image via the host:

```bash
# on host
podman save -o /tmp/bootc.tar ghcr.io/<registry>/<image>:latest
podman cp /tmp/bootc.tar ov-openclaw-desktop:/tmp/bootc.tar
# inside the pod
podman load -i /tmp/bootc.tar                              # nested rootless store
podman --connection ov-root load -i /tmp/bootc.tar         # podman-machine rootful store
ov vm build <image> --transport containers-storage
```

### Docker Hub rate-limit note

For nested test harnesses, prefer `quay.io/libpod/alpine:latest` (unmetered)
over `docker.io/library/alpine` — the baked `nested-podman-run` eval already
uses this mirror.

## What works from inside the desktop

Every `ov` verb family runs as uid 1000 inside the container sandbox:
`ov box build/generate/validate/merge/inspect/list/pull`,
`ov eval box/live/cdp/wl/dbus/vnc/mcp`,
`ov config/deploy/start/stop/update/remove/shell/cmd/service/status/logs`,
`ov vm list/create/start/stop/ssh/destroy` (rootless libvirt session),
`ov doctor/secrets/settings/alias`. The `ov` layer bakes only the binary — for
build-mode verbs that read `box.yml`, mount or `podman cp` the project in.

## Volumes

- `chrome-data` → `~/.chrome-debug` (Chrome profile, from the chrome layer)
- `selkies-config` → `~/.config/selkies`
- `models` → `~/.ollama` (Ollama model storage)

## Verification

```bash
ov status openclaw-desktop                       # all services RUNNING
curl -k https://localhost:3000                   # selkies HTTPS 200
curl -s http://localhost:18789                   # openclaw gateway
curl -s http://localhost:11434/api/tags          # ollama API
ov eval wl screenshot openclaw-desktop t.png     # desktop screenshot
ov shell openclaw-desktop -c 'podman run --rm quay.io/libpod/alpine:latest /bin/true'
```

The baked image-level `eval:` carries the nested-rootless posture checks (subuid
two-ranges, `newuidmap` cap, `policy.json`, containers.conf `userns=host`,
`_CONTAINERS_USERNS_CONFIGURED` + `BUILDAH_ISOLATION` env), the deploy-scope
nested-toolchain checks (nested `podman run`, `virsh` session list, in-container
`ov version`/`ov doctor`), and the three fused services' liveness (gateway,
ollama API, chrome-devtools-mcp port). The R10 bed is
`eval-openclaw-desktop-pod` (`ov eval run eval-openclaw-desktop-pod`).

## Key Layers

- `/ov-selkies:selkies-desktop-layer` — the streaming desktop metalayer
- `/ov-openclaw:openclaw-full` — gateway + 27 tools (claude-code/codex/gemini)
- `/ov-ollama:ollama` — CPU/GPU-agnostic Ollama layer (GPU is image-level)
- `/ov-tools:ov` — the full toolchain: ov binary + virtualization + gocryptfs + socat
- `/ov-distros:container-nesting` — rootless nested podman recipe (RCA for `unmask=/proc/*`)
- `/ov-infrastructure:virtualization` — supervisord-managed virtqemud/virtnetworkd
- `/ov-distros:agent-forwarding` — GPG/SSH/direnv agent sockets

## Related Images

- `/ov-openclaw:openclaw-full` — the headless gateway + tools WITHOUT the desktop / ollama / ov toolchain.
- `/ov-openclaw:openclaw` — minimal gateway only.
- `/ov-selkies:selkies-labwc` — the CPU streaming desktop WITHOUT openclaw / ollama / ov toolchain.
- `/ov-selkies:selkies-labwc-nvidia` — GPU streaming desktop (base nvidia), no openclaw/ollama/ov toolchain.
- `/ov-distros:fedora-ov` / `/ov-coder:arch-ov` — root-mode ov toolchain WITHOUT a streaming desktop.

## When to Use This Skill

**MUST be invoked** when the task involves:

- Building, deploying, or troubleshooting the `openclaw-desktop` image.
- Running `ov` verbs (image build, nested pods, rootless VMs) from inside a
  streaming desktop.
- The OpenClaw gateway, AI CLIs, or a CPU Ollama running alongside a Wayland
  streaming desktop in one image.
- The non-`--privileged` rootless-in-rootless nesting path on a CachyOS desktop.

## Related

- `/ov-image:image` — image family umbrella (`image:` entries, build/validate/inspect/list)
- `/ov-eval:eval` — declarative testing + the `eval-openclaw-desktop-pod` R10 bed
- `/ov-eval:cdp`, `/ov-eval:wl` — desktop automation on this image
- `/ov-core:ov-config` — deploy setup (tunnel, port remapping, multi-instance, encrypted volumes)
