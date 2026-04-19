---
name: selkies-desktop-ov
description: |
  Fully rootless Selkies streaming desktop + the full ov toolchain inside
  one image. Runs as uid 1000 with zero --privileged / zero cap_add; nested
  rootless podman and rootless libvirt session VMs work via config-level
  namespace sharing and the surgical unmask=/proc/* security_opt.
  Use when working with the selkies-desktop-ov image — especially for any
  "browser-accessible desktop that can itself build images, start pods, or
  launch VMs" workflow.
---

# selkies-desktop-ov

The non-root counterpart to `fedora-ov`/`arch-ov`: a browser-accessible
Wayland streaming desktop with the full `ov` toolchain inside, all running
as uid 1000 `user` with the same posture as the production
`selkies-desktop` service. Every `ov` verb (image build, start/stop/config,
nested podman, rootless libvirt VMs, gocryptfs encrypted volumes, MCP
probing) works from a terminal inside the streaming desktop.

## Definition

```yaml
selkies-desktop-ov:
  base: nvidia                 # preserves NVENC + CUDA + CDI
  layers:
    - agent-forwarding
    - selkies-desktop          # full streaming desktop stack
    - ov-full                  # ov binary + virtualization + gocryptfs + socat
    - container-nesting        # nested rootless podman/buildah/skopeo
    - golang
    - gh
    - dbus
    - ov
  ports:
    - "3000:3000"              # Selkies web UI (Traefik HTTPS)
    - "9222:9222"              # Chrome DevTools Protocol
    - "9224:9224"              # chrome-devtools-mcp (Streamable HTTP)
    - "2222:2222"              # sshd-wrapper
  platforms:
    - linux/amd64
```

No `uid:`/`gid:`/`user:`/`network:` override — inherits `1000/1000/user`
and the default `ov` bridge network. Rootless is the whole design.

## Resolved security posture (OCI label)

| Field | Value |
|---|---|
| `cap_add` | **(empty)** |
| `security_opt` | `[unmask=/proc/*]` (from `/ov-layers:container-nesting`) |
| `devices` | `[/dev/fuse, /dev/net/tun]` (from `/ov-layers:container-nesting`) |
| `shm_size` | `1g` (from `/ov-layers:chrome`) |
| `memory_max` | `6g` (from `/ov-layers:chrome`) |
| `privileged` | `false` |
| Network | default ov bridge (NOT host) |
| UID / user | `1000 / user` |

**No `--privileged`. No `cap_add: ALL`. No `seccomp=unconfined`. No
`label=disable`.** The only security relaxation is `unmask=/proc/*` —
surgical, documented in `/ov-layers:container-nesting` with the full
kernel-level RCA.

## Difference from sibling selkies images

| Image | Base | Extra layers vs selkies-desktop-nvidia | Security | Purpose |
|---|---|---|---|---|
| `selkies-desktop` | `fedora-nonfree` | — | zero-cap baseline | CPU-encoded streaming desktop |
| `selkies-desktop-nvidia` | `nvidia` | — | zero-cap baseline | NVENC streaming desktop |
| **`selkies-desktop-ov`** | `nvidia` | **ov-full + container-nesting + golang + gh** | `unmask=/proc/*` + `/dev/fuse,/dev/net/tun` | Streaming desktop that can build images + run nested pods + launch VMs |
| `selkies-desktop-bootc` | `fedora-bootc:43` | tailscale + keepassxc + bootc-base + rpmfusion | systemd on a bootc VM | Bootable streaming desktop VM |

The `ov-full` layer transitively adds `ov` + `virtualization` (now
ships `virtqemud` + `virtnetworkd` supervisord programs) + `gocryptfs` +
`socat` + `podman-machine` + `gvisor-tap-vsock`. The `container-nesting`
layer adds `buildah` + `fuse-overlayfs` + `skopeo` + `tailscale` +
`libsecret` + the dual-location containers.conf / storage.conf /
policy.json configs + the two canonical env vars
(`_CONTAINERS_USERNS_CONFIGURED=""` + `BUILDAH_ISOLATION=chroot`).

## Nested rootless podman — how it works here

See `/ov-layers:container-nesting` for the full kernel-level RCA. The
short version:

1. Podman's default rootless outer container injects
   `linux.maskedPaths` covering `/proc/kcore`, `/proc/keys`, etc.
2. The kernel's `fs/namespace.c:mount_too_revealing()` then refuses any
   fresh `/proc` mount from that container's namespace (including from
   an inner crun creating an alpine container).
3. `unmask=/proc/*` tells podman NOT to emit those masks on the outer,
   so the inner `/proc` mount has nothing to mismatch with.

Empirically verified on this host (2026-04-19) against
`quay.io/libpod/alpine:latest`:

```bash
ov shell selkies-desktop-ov -c 'podman run --rm quay.io/libpod/alpine:latest /bin/true'
# NESTED_ROOTLESS_OK — zero extra caps
```

With the image's default bundle + no additional CLI flags, nested
`podman run` and `podman build` both succeed.

## Nested rootless VMs — how it works here

See `/ov-layers:virtualization` for the full story. The short version:

- `virtqemud --timeout 0` runs as a supervisord program (priority 5),
  inheriting uid 1000 from the parent supervisord.
- `virtnetworkd --timeout 0` runs the same way (priority 6).
- Libvirt `qemu:///session` mode keys off `$XDG_RUNTIME_DIR` and
  doesn't need CAP_SYS_ADMIN or root. `/dev/kvm` is the only device
  needed.

```bash
ov shell selkies-desktop-ov -c 'virsh -c qemu:///session domcapabilities | grep "<domain>"'
# → <domain>kvm</domain>  (HW acceleration confirmed)
```

## Ports

| Port | Service | Identical to |
|---|---|---|
| 3000 | Selkies web UI (Traefik HTTPS, self-signed) | `/ov-images:selkies-desktop-nvidia` |
| 9222 | Chrome DevTools Protocol (via `cdp-proxy`) | `/ov-images:selkies-desktop-nvidia` |
| 9224 | `chrome-devtools-mcp` (Streamable HTTP) | `/ov-images:selkies-desktop-nvidia` |
| 2222 | `sshd-wrapper` (supervisord-managed) | `/ov-images:selkies-desktop-nvidia` |

For multi-instance deployments alongside a running `selkies-desktop*`
instance (which already holds those host ports), remap via
`ov config selkies-desktop-ov -p 3010:3000 -p 9232:9222 -p 9242:9224 -p 2232:2222`
or similar. See `/ov:config` for the `-p` flag and `/ov-layers:selkies-desktop`
for the multi-instance proxy deployment pattern.

## What works from inside the desktop

Every `ov` verb family, verified empirically:

| Verb family | Notes |
|---|---|
| `ov version`, `ov doctor` | `/dev/kvm` + libvirt socket reported present |
| `ov image build/generate/validate/merge/inspect/list/pull/test` | Build-mode — nested podman handles the actual image build |
| `ov test run/cdp/wl/dbus/vnc/mcp`, `ov image test` | Test-mode |
| `ov config/deploy/start/stop/update/remove/shell/cmd/service/status/logs/tmux` | Deploy-mode — each spawns a nested container under the inner podman |
| `ov vm list/create/start/stop/ssh/destroy` | Rootless libvirt session via `virtqemud` |
| `ov doctor/udev/record/secrets/settings/alias` | Host-adjacent verbs |

All run as uid 1000 inside the container sandbox. From the host's
perspective, the container never escalates — it stays pinned to the
invoking user (via `ov shell`'s `--userns=keep-id:uid=1000,gid=1000`).

## Empirical test results (2026-04-19)

- `ov image test selkies-desktop-ov` — **91 passed · 0 failed · 0 skipped**
- `ov image inspect selkies-desktop-ov` — resolved `security` label matches
  the table above (cap_add empty, security_opt: `[unmask=/proc/*]`,
  devices: `[/dev/fuse, /dev/net/tun]`).
- Nested smoke: `podman run --rm quay.io/libpod/alpine:latest true`
  succeeds with no CLI overrides.
- VM smoke: `virsh -c qemu:///session domcapabilities` reports
  `<domain>kvm</domain>`.

Full RCA of the path from `--privileged` to the surgical
`unmask=/proc/*` fix lives in `/ov-layers:container-nesting`. The plan
file that drove this implementation documents every test variation
against `quay.io/podman/stable` for comparison.

## Docker Hub rate-limit note

Nested `podman run docker.io/library/alpine:3` is rate-limited when
many fresh outer containers each re-pull alpine. For test harnesses
inside selkies-desktop-ov, use `quay.io/libpod/alpine:latest` (unmetered)
instead — the baked `nested-podman-run` image-scope test already uses
this mirror.

## Deploy notes

- Encrypted-volume workflow via `/ov:config --encrypt`: works because
  `gocryptfs` (from `ov-full`) + `/dev/fuse` (from `container-nesting`)
  are both baked in.
- Tunnel (Tailscale/Cloudflare) configuration lives in `deploy.yml`
  only, same rule as all other selkies-desktop variants. See `/ov:deploy`
  (Instance Tunnel Inheritance).
- Multi-instance deployments get auto-disambiguated MCP server names
  (`chrome-devtools-<instance>`) just like `/ov-images:selkies-desktop`.

## Security trade-offs (honest)

- **`unmask=/proc/*`** exposes `/proc/kcore` and `/proc/keys` on the
  outer container. Within a rootless user namespace, actually reading
  those still requires `CAP_SYS_ADMIN` in the init userns (which rootless
  containers never have). The information leak is minimal; the
  alternative is `--privileged`, which is a much bigger hammer.
- **`userns=host`** in containers.conf means nested containers share the
  outer's user namespace. This is the canonical Red Hat rootless-in-rootless
  pattern (matches `quay.io/podman/stable`). Strictly less isolated than
  "each nested container gets its own userns", but the alternative is
  blocked by the kernel's `mount_too_revealing()` check.
- **Every process still runs as uid 1000 on the host.** No host
  privilege escalation. No access to other users' files. No kernel
  module loading. `--privileged` rootless wouldn't grant those either.

## Key Layers

- `/ov-layers:selkies-desktop` — the full streaming desktop metalayer (Chrome, labwc, pipewire, waybar, selkies, wl-tools, pavucontrol, swaync, sshd, etc.)
- `/ov-layers:ov-full` — composition: ov + virtualization + gocryptfs + socat + podman-machine + gvisor-tap-vsock
- `/ov-layers:container-nesting` — the rootless nested podman recipe (authoritative RCA for `mount_too_revealing()` + `unmask=/proc/*`)
- `/ov-layers:virtualization` — QEMU/KVM packages + supervisord-managed virtqemud/virtnetworkd for rootless session libvirt
- `/ov-layers:nvidia` — GPU runtime (via the `nvidia` base image)
- `/ov-layers:cuda` — CUDA toolkit (via the `nvidia` base image)
- `/ov-layers:dbus` — session bus for desktop services
- `/ov-layers:agent-forwarding` — GPG/SSH/direnv agent sockets

## Related Images

- `/ov-images:selkies-desktop-nvidia` — same streaming desktop, WITHOUT the ov toolchain (no nested podman, no VMs). Prefer this variant if you just need a GPU-accelerated browser desktop.
- `/ov-images:selkies-desktop` — CPU-encoded variant on `fedora-nonfree`.
- `/ov-images:selkies-desktop-bootc` — bootable VM flavor of the streaming desktop (no ov toolchain).
- `/ov-images:fedora-ov` — root-mode ov toolchain WITHOUT the streaming desktop (traditional `cap_add:[ALL] + host networking` posture).
- `/ov-images:arch-ov` — Arch Linux counterpart of fedora-ov.
- `/ov-images:nvidia` — the parent base image (CUDA + NVIDIA runtime).

## Related Commands

- `/ov:shell` — open an interactive shell inside the container as uid 1000
- `/ov:config` — deploy setup (tunnel, port remapping, multi-instance, encrypted volumes)
- `/ov:test` — parent router for live-container verbs (`ov test cdp|wl|dbus|vnc|mcp`)
- `/ov:vm` — nested libvirt VM lifecycle (works via `virtqemud` session)
- `/ov:build` — building other images from inside this one (nested podman)
- `/ov:cdp` — drive the baked-in Chrome via DevTools Protocol
- `/ov:wl` — Wayland automation (screenshots, input, windows, clipboard)
- `/ov:dbus` — D-Bus notifications via in-container ov
- `/ov:mcp` — probes the `chrome-devtools-mcp` server on port 9224 (29 tools) and any other MCP server declared by co-deployed images

## When to Use This Skill

**MUST be invoked** when the task involves:

- Building, deploying, or troubleshooting the `selkies-desktop-ov` image.
- Running `ov` verbs from inside a streaming desktop (e.g., "I want to
  `ov image build` something while watching the build in the browser").
- Nested rootless podman or rootless libvirt VMs from a non-root
  container (the kernel-level RCA is in `/ov-layers:container-nesting`
  and `/ov-layers:virtualization`; the composition that puts them
  together is here).
- Any discussion of the non-`--privileged` path for rootless-in-rootless
  nesting — this is the canonical worked example.
