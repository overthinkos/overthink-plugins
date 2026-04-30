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
| `security_opt` | `[unmask=/proc/*]` (from `/ov-foundation:container-nesting`) |
| `devices` | `[/dev/fuse, /dev/net/tun]` (from `/ov-foundation:container-nesting`) |
| `shm_size` | `1g` (from `/ov-selkies:chrome`) |
| `memory_max` | `6g` (from `/ov-selkies:chrome`) |
| `privileged` | `false` |
| Network | default ov bridge (NOT host) |
| UID / user | `1000 / user` |

**No `--privileged`. No `cap_add: ALL`. No `seccomp=unconfined`. No
`label=disable`.** The only security relaxation is `unmask=/proc/*` —
surgical, documented in `/ov-foundation:container-nesting` with the full
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

See `/ov-foundation:container-nesting` for the full kernel-level RCA. The
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

See `/ov-foundation:virtualization` for the full story. The short version:

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
| 3000 | Selkies web UI (Traefik HTTPS, self-signed) | `/ov-selkies:selkies-desktop-nvidia` |
| 9222 | Chrome DevTools Protocol (via `cdp-proxy`) | `/ov-selkies:selkies-desktop-nvidia` |
| 9224 | `chrome-devtools-mcp` (Streamable HTTP) | `/ov-selkies:selkies-desktop-nvidia` |
| 2222 | `sshd-wrapper` (supervisord-managed) | `/ov-selkies:selkies-desktop-nvidia` |

For multi-instance deployments alongside a running `selkies-desktop*`
instance (which already holds those host ports), remap via
`ov config selkies-desktop-ov -p 3010:3000 -p 9232:9222 -p 9242:9224 -p 2232:2222`
or similar. See `/ov-core:config` for the `-p` flag and `/ov-selkies:selkies-desktop`
for the multi-instance proxy deployment pattern.

## What works from inside the desktop

Every `ov` verb family, verified empirically:

| Verb family | Notes |
|---|---|
| `ov version`, `ov doctor` | `/dev/kvm` + libvirt socket reported present |
| `ov image build/generate/validate/merge/inspect/list/pull/test` | Build-mode — nested podman handles the actual image build |
| `ov eval live/cdp/wl/dbus/vnc/mcp`, `ov eval image` | Test-mode |
| `ov config/deploy/start/stop/update/remove/shell/cmd/service/status/logs/tmux` | Deploy-mode — each spawns a nested container under the inner podman |
| `ov vm list/create/start/stop/ssh/destroy` | Rootless libvirt session via `virtqemud` |
| `ov doctor/udev/record/secrets/settings/alias` | Host-adjacent verbs |

All run as uid 1000 inside the container sandbox. From the host's
perspective, the container never escalates — it stays pinned to the
invoking user (via `ov shell`'s `--userns=keep-id:uid=1000,gid=1000`).

## Empirical test results (2026-04-19)

- `ov eval image ghcr.io/overthinkos/selkies-desktop-ov:latest` — **91 passed · 0 failed · 0 skipped**.
- `ov test selkies-desktop-ov -i test` (live service, port-remapped) — **118 passed · 0 failed · 0 skipped** (adds deploy-scope verification: nested podman alpine, libvirt session list, KVM domcaps, in-container `ov version` / `ov doctor`).
- `ov image inspect selkies-desktop-ov` — resolved `security` label matches
  the table above (cap_add empty, security_opt: `[unmask=/proc/*]`,
  devices: `[/dev/fuse, /dev/net/tun]`).

### Two-level nested-virtualization proof (end-to-end `ov vm` run-through)

Verified on 2026-04-19: the full `ov vm build/create/ssh/stop/destroy` lifecycle completes inside the rootless pod, **two levels of KVM nesting deep**:

1. Host (rootless podman, uid 1000) runs `ov-selkies-desktop-ov-test`.
2. Inside that container, `ov vm build selkies-desktop-bootc --transport containers-storage` auto-falls back to `engine.rootful=machine` (no host `sudo` in reach), which spawns a **podman-machine VM (nested VM #1)** via KVM passthrough, mounts `/home/user`, and runs `bootc install to-disk`.
3. `ov vm create selkies-desktop-bootc -i smoke --ram 2G --cpus 2 --ssh-key generate` then boots the produced qcow2 as a QEMU user-net VM (**nested VM #2**).
4. `ov vm ssh selkies-desktop-bootc -i smoke -- uname -a` returns `Linux fedora 6.19.12-200.fc43.x86_64` from the guest; `/etc/os-release` reports Fedora Linux 43, rootfs is composefs-on-overlay with `/dev/vda3` ext4.
5. `ov vm destroy --disk` cleans up fully.

No `--privileged`, no `cap_add`, no seccomp relaxations beyond the baked `[unmask=/proc/*]`. Only `/dev/kvm` passthrough.

### Cross-storage loading for private bootc images

The podman-machine spawned by step 2 above has its **own rootful storage**, separate from the outer container's nested rootless storage. Pulling a private bootc image from the registry (403 Forbidden without creds) can be sidestepped by loading the image via the host:

```bash
# on host
podman save -o /tmp/bootc.tar ghcr.io/overthinkos/selkies-desktop-bootc:latest
podman cp /tmp/bootc.tar ov-selkies-desktop-ov-test:/tmp/bootc.tar

# inside the pod
podman load -i /tmp/bootc.tar                              # into nested rootless store
podman save -o /tmp/bootc.tar ghcr.io/overthinkos/selkies-desktop-bootc:latest
podman --connection ov-root load -i /tmp/bootc.tar         # into podman-machine rootful store
ov vm build selkies-desktop-bootc --transport containers-storage
```

Full RCA of the path from `--privileged` to the surgical
`unmask=/proc/*` fix lives in `/ov-foundation:container-nesting`. The plan
file that drove this implementation documents every test variation
against `quay.io/podman/stable` for comparison.

### Full supervisord program roster on `ov-selkies-desktop-ov-test`

All 15 programs RUNNING under uid 1000 on a healthy container (2026-04-19):

```
cdp-proxy            chrome               chrome-crash
chrome-devtools-mcp  dbus                 labwc
pipewire             selkies              selkies-fileserver
sshd                 swaync               traefik
virtnetworkd (pid 7, priority 6)    virtqemud (pid 6, priority 5)
waybar
```

Check anytime with `ov cmd selkies-desktop-ov -i <instance> 'supervisorctl status'`.

### Using `ov` from inside the pod

The `ov` layer bakes **only the binary**, not `image.yml` / `build.yml` / `layers/` / `plugins/`. These commands work standalone inside the container:

- `ov version`, `ov doctor`, `ov settings list`, `ov alias list`, `ov secrets list`, `ov vm list`, `ov udev list`.

These need the repo mounted or `podman cp`'d in:

- `ov image validate / list / inspect / build / merge / generate / pull / test`, `ov vm build` (reads `image.yml` for the bootc ref).

Quick import: `podman cp image.yml <container>:/home/user/image.yml && podman cp build.yml <container>:/home/user/ && podman cp layers <container>:/home/user/ && podman cp plugins <container>:/home/user/` — then `cd /home/user && ov image build <target>`.

## Docker Hub rate-limit note

Nested `podman run docker.io/library/alpine:3` is rate-limited when
many fresh outer containers each re-pull alpine. For test harnesses
inside selkies-desktop-ov, use `quay.io/libpod/alpine:latest` (unmetered)
instead — the baked `nested-podman-run` image-scope test already uses
this mirror.

## Deploy notes

- Encrypted-volume workflow via `/ov-core:config --encrypt`: works because
  `gocryptfs` (from `ov-full`) + `/dev/fuse` (from `container-nesting`)
  are both baked in.
- Tunnel (Tailscale/Cloudflare) configuration lives in `deploy.yml`
  only, same rule as all other selkies-desktop variants. See `/ov-core:deploy`
  (Instance Tunnel Inheritance).
- Multi-instance deployments get auto-disambiguated MCP server names
  (`chrome-devtools-<instance>`) just like `/ov-selkies:selkies-desktop`.

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

- `/ov-selkies:selkies-desktop` — the full streaming desktop metalayer (Chrome, labwc, pipewire, waybar, selkies, wl-tools, pavucontrol, swaync, sshd, etc.)
- `/ov-coder:ov-full` — composition: ov + virtualization + gocryptfs + socat + podman-machine + gvisor-tap-vsock
- `/ov-foundation:container-nesting` — the rootless nested podman recipe (authoritative RCA for `mount_too_revealing()` + `unmask=/proc/*`)
- `/ov-foundation:virtualization` — QEMU/KVM packages + supervisord-managed virtqemud/virtnetworkd for rootless session libvirt
- `/ov-foundation:nvidia` — GPU runtime (via the `nvidia` base image)
- `/ov-foundation:cuda` — CUDA toolkit (via the `nvidia` base image)
- `/ov-foundation:dbus` — session bus for desktop services
- `/ov-foundation:agent-forwarding` — GPG/SSH/direnv agent sockets

## Related Images

- `/ov-selkies:selkies-desktop-nvidia` — same streaming desktop, WITHOUT the ov toolchain (no nested podman, no VMs). Prefer this variant if you just need a GPU-accelerated browser desktop.
- `/ov-selkies:selkies-desktop` — CPU-encoded variant on `fedora-nonfree`.
- `/ov-selkies:selkies-desktop-bootc` — bootable VM flavor of the streaming desktop (no ov toolchain).
- `/ov-foundation:fedora-ov` — root-mode ov toolchain WITHOUT the streaming desktop (traditional `cap_add:[ALL] + host networking` posture).
- `/ov-coder:arch-ov` — Arch Linux counterpart of fedora-ov.
- `/ov-foundation:nvidia` — the parent base image (CUDA + NVIDIA runtime).

## Related Commands

- `/ov-core:shell` — open an interactive shell inside the container as uid 1000
- `/ov-core:config` — deploy setup (tunnel, port remapping, multi-instance, encrypted volumes)
- `/ov-build:eval` — parent router for live-container verbs (`ov eval cdp|wl|dbus|vnc|mcp`)
- `/ov-advanced:vm` — nested libvirt VM lifecycle (works via `virtqemud` session)
- `/ov-build:build` — building other images from inside this one (nested podman)
- `/ov-advanced:cdp` — drive the baked-in Chrome via DevTools Protocol
- `/ov-advanced:wl` — Wayland automation (screenshots, input, windows, clipboard)
- `/ov-advanced:dbus` — D-Bus notifications via in-container ov
- `/ov-build:mcp` — probes the `chrome-devtools-mcp` server on port 9224 (29 tools) and any other MCP server declared by co-deployed images

## When to Use This Skill

**MUST be invoked** when the task involves:

- Building, deploying, or troubleshooting the `selkies-desktop-ov` image.
- Running `ov` verbs from inside a streaming desktop (e.g., "I want to
  `ov image build` something while watching the build in the browser").
- Nested rootless podman or rootless libvirt VMs from a non-root
  container (the kernel-level RCA is in `/ov-foundation:container-nesting`
  and `/ov-foundation:virtualization`; the composition that puts them
  together is here).
- Any discussion of the non-`--privileged` path for rootless-in-rootless
  nesting — this is the canonical worked example.

## Related

- `/ov-build:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
