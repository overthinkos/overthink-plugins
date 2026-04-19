---
name: virtualization
description: |
  QEMU/KVM/libvirt stack plus supervisord-managed virtqemud and virtnetworkd
  daemons. Works rootless: `qemu:///session` mode serves uid 1000 with only
  /dev/kvm passthrough — no SYS_ADMIN, no root. Canonical source for the
  rootless nested VM recipe.
  Use when working with virtual machines, QEMU, KVM, libvirt, or the
  virtualization layer.
---

# virtualization — QEMU/KVM/libvirt with rootless session daemons

## Overview

Installs the QEMU/KVM/libvirt stack AND ships supervisord programs for
`virtqemud` + `virtnetworkd` running as the layer consumer's uid. In
session mode (`qemu:///session`), these daemons and their clients all
work at uid 1000 with only `/dev/kvm` device passthrough — no
`CAP_SYS_ADMIN`, no root escalation. Makes nested rootless VM creation
work end-to-end from a rootless outer container.

## Layer Properties

| Property | Value |
|----------|-------|
| `depends:` | `supervisord` |
| Services registered | `virtqemud` (priority 5), `virtnetworkd` (priority 6) |
| Devices | `/dev/kvm` is required by consumers; declared at the image level or via `/ov-layers:container-nesting` |

## Packages (RPM)

Baseline QEMU + libvirt client + guest tools:

- `bindfs`, `genisoimage`, `guestfs-tools`, `libguestfs-tools`
- `libvirt-nss`, `passt`, `qemu-guest-agent`
- `qemu-img`, `qemu-kvm`
- `qemu-system-aarch64`, `qemu-system-arm`, `qemu-system-x86-core`
- `tigervnc`, `virt-manager`, `virt-viewer`

Libvirt split-daemon set (explicitly listed so images can run the
session daemon even without `virt-manager`):

- `libvirt-client` — `virsh` binary
- `libvirt-daemon` — `/usr/sbin/virtqemud`, `/usr/sbin/virtnetworkd`
- `libvirt-daemon-config-network`
- `libvirt-daemon-driver-qemu`

## Packages (Pacman)

`qemu-full`, `qemu-img`, `virtiofsd`, `libvirt`.

## Supervisord services

The layer's `service:` block registers two programs:

```ini
[program:virtqemud]
command=/usr/sbin/virtqemud --timeout 0
autostart=true
autorestart=true
priority=5
startsecs=2
stdout_logfile=/dev/fd/1
stdout_logfile_maxbytes=0
redirect_stderr=true

[program:virtnetworkd]
command=/usr/sbin/virtnetworkd --timeout 0
autostart=true
autorestart=true
priority=6
startsecs=2
stdout_logfile=/dev/fd/1
stdout_logfile_maxbytes=0
redirect_stderr=true
```

Two deliberate choices:

- **`--timeout 0`** keeps both daemons in the foreground (they would
  otherwise self-exit on idle). Required for supervisord to manage
  them.
- **No `user=` directive.** The supervisord programs inherit the parent
  supervisord's uid. On `fedora-ov`/`arch-ov`/`githubrunner`,
  supervisord runs as uid 0 → daemons run as uid 0 →
  `qemu:///session` targets root's session. On `selkies-desktop-ov`,
  supervisord runs as uid 1000 → daemons run as uid 1000 →
  `qemu:///session` targets user's session. Both work because the
  session URI keys off `$XDG_RUNTIME_DIR` rather than a fixed socket
  path.

Priority 5/6 places both daemons ahead of every desktop program
(labwc=12, selkies=18) so the session socket is available by the time
any later service or shell tries to connect.

See `/ov:generate` for the supervisord `fragment_assembly` init model
that emits these program blocks into `.build/<image>/supervisor/NN-virtualization.conf`.

## Rootless libvirt — `qemu:///session`

`ov/vm.go:22` already hardcodes `qemu:///session` as the ov default.
This layer makes that URI actually work inside a container at uid
1000:

- No `CAP_SYS_ADMIN` required on the outer container.
- No `security_opt` relaxation beyond what `/ov-layers:container-nesting` contributes (and even that isn't needed for VMs alone).
- Only `/dev/kvm` must be passed through. On the host, it's typically
  `crw-rw-rw-` (world-readable/writable), so any uid that can `open()` it
  gets KVM acceleration.
- `virsh -c qemu:///session domcapabilities` reports
  `<domain>kvm</domain>` once the daemons are running — empirically
  verified (2026-04-19) on `selkies-desktop-ov` at uid 1000.

## Tests baked into the layer

**Build-scope (run by `ov image test <image>` without deploying):**

- `virtqemud-binary` — `/usr/sbin/virtqemud` exists.
- `virtnetworkd-binary` — `/usr/sbin/virtnetworkd` exists.
- `virsh-binary` — `/usr/bin/virsh` exists.
- `qemu-system-x86-binary` — `/usr/bin/qemu-system-x86_64` exists.
- `qemu-img-binary` — `/usr/bin/qemu-img` exists.

**Deploy-scope (run against a live `ov start`ed container):**

- `virtqemud-running` — supervisorctl reports RUNNING.
- `virtnetworkd-running` — supervisorctl reports RUNNING.
- `libvirt-session-list` — `virsh -c qemu:///session list --all` exits 0.
- `libvirt-kvm-acceleration` — `virsh -c qemu:///session domcapabilities | grep -c '<domain>kvm</domain>'` ≥ 1.

## Usage

```yaml
# image.yml — rootless nested VM image
selkies-desktop-ov:
  layers:
    - ...
    - ov-full             # transitively pulls virtualization
    - container-nesting   # donates /dev/fuse + /dev/net/tun devices (VMs only need /dev/kvm)
```

`/dev/kvm` is auto-detected at `ov shell`/`ov start` time by
`ov/devices.go` (scans `/dev/kvm`, `/dev/fuse`, `/dev/dri/renderD*`,
`/dev/net/tun`, `/dev/vhost-*`, `/dev/hwrng`) — no image-level
`security.devices:` entry needed for the typical deployment.

## Composition chain

`ov-full` → composes `virtualization` (this layer) + `ov` + `gocryptfs` +
`socat` + `podman-machine` + `gvisor-tap-vsock`. So any image that pulls
`ov-full` automatically gets the supervisord-managed virtqemud/virtnetworkd
programs.

## Used In Images

- `/ov-images:selkies-desktop-ov` — rootless VM host inside a streaming desktop
- `/ov-images:fedora-ov` — root VM host (same daemons, uid 0)
- `/ov-images:arch-ov` — Arch counterpart
- `/ov-images:githubrunner` — VMs for CI workloads
- `/ov-images:aurora`, `/ov-images:bazzite-ai` — bootc siblings (currently `enabled: false`)

## Related Layers

- `/ov-layers:ov-full` — composition that pulls this layer into ov-toolchain images
- `/ov-layers:container-nesting` — pairs with this layer for images that need both nested containers AND nested VMs; also donates `/dev/kvm`-adjacent devices
- `/ov-layers:socat` — part of `ov-full` alongside virtualization; used for VM console/hostfwd relays
- `/ov-layers:gocryptfs` — part of `ov-full`; for encrypting VM disk storage

## Related Commands

- `/ov:vm` — VM lifecycle (build, create, start, stop, ssh, console, destroy); defaults to `qemu:///session` at ov/vm.go:22
- `/ov:generate` — Containerfile generation; emits the supervisord `NN-virtualization.conf` fragment via `fragment_assembly` init model
- `/ov:layer` — layer authoring reference (tasks, vars, service blocks, tests syntax)
- `/ov:test` — declarative testing framework for the layer's `tests:` block (file, service, command verbs)

## When to Use This Skill

**MUST be invoked** when:

- Adding VM support to any image (this layer donates the daemons + packages).
- Debugging `virsh -c qemu:///session` connection failures — the two
  most common causes are supervisord not yet started the daemons (check
  `supervisorctl status virtqemud`) or `/dev/kvm` not passed through
  (check `ls -la /dev/kvm` inside the container).
- Running `ov vm` from inside a rootless container (the
  supervisord-managed daemons here are what makes that work).
- Understanding why the layer no longer just installs packages — the
  2026-04-19 refactor added the supervisord programs + `depends:
  supervisord` + the deploy-scope tests, turning it from a
  package-only layer into a service provider.
