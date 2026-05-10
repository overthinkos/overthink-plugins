---
name: virtualization
description: |
  QEMU/KVM/libvirt stack — works identically under supervisord (containers/
  pods, custom `exec:` form) AND under systemd (host installs / bootc / VMs,
  use_packaged: virtqemud.socket / virtnetworkd.socket). Uses the mixed-entry
  `service:` schema (CLAUDE.md "Init-system polymorphism") — same name
  appears twice in the service: list, init system at deploy time picks the
  matching form. Canonical worked example of the polymorphism pattern.
---

# virtualization — QEMU/KVM/libvirt for both supervisord and systemd init systems

## Canonical worked example: mixed-`service:` polymorphism (2026-05)

This layer is the canonical demonstration that ONE layer can serve BOTH container/pod targets (supervisord init) AND host/bootc/VM targets (systemd init) without spawning a `<name>-host` sibling. The mechanism is the **mixed `service:` schema pattern**: each daemon (`virtqemud`, `virtnetworkd`) appears TWICE in the layer's `service:` list — once with `use_packaged: <unit>.socket` (rendered only on systemd-init targets) and once with custom `exec:` (rendered only on supervisord-init targets). The init system at deploy time picks the matching form; the other entry is silently skipped.

```yaml
service:
  # virtqemud — systemd render
  - name: virtqemud
    use_packaged: virtqemud.socket
    enable: true
    scope: system
  # virtqemud — supervisord render
  - name: virtqemud
    exec: "/usr/sbin/virtqemud --timeout 0"
    restart: always
    priority: 5
    start_secs: 2
    enable: true
    scope: system
  # virtnetworkd — same mixed-entry pattern
  - name: virtnetworkd
    use_packaged: virtnetworkd.socket
    enable: true
    scope: system
  - name: virtnetworkd
    exec: "/usr/sbin/virtnetworkd --timeout 0"
    restart: always
    priority: 6
    start_secs: 2
    enable: true
    scope: system
```

Why this matters: the previous design had a `virtualization-host` sibling layer that duplicated package lists, eval probes, and tasks for systemd targets. Drift between the two siblings was inevitable. The mixed-entry pattern eliminates the sibling — ONE layer covers both contexts; the schema does the polymorphism. The 2026-05 polymorphism cutover deleted the `-host` sibling along with `ov-full-host`. See CLAUDE.md "Init-system polymorphism via mixed `service:` entries" for the rule and `/ov-image:layer` "Service Declaration" → "Anti-pattern: `<name>-host` / `<name>-pod` sibling layers" for what NOT to do.

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
| `require:` | `supervisord` |
| Services registered | `virtqemud` (priority 5), `virtnetworkd` (priority 6) |
| Devices | `/dev/kvm` is required by consumers; declared at the image level or via `/ov-distros:container-nesting` |

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

See `/ov-build:generate` for the supervisord `fragment_assembly` init model
that emits these program blocks into `.build/<image>/supervisor/NN-virtualization.conf`.

## Rootless libvirt — `qemu:///session`

`ov/vm.go:22` already hardcodes `qemu:///session` as the ov default.
This layer makes that URI actually work inside a container at uid
1000:

- No `CAP_SYS_ADMIN` required on the outer container.
- No `security_opt` relaxation beyond what `/ov-distros:container-nesting` contributes (and even that isn't needed for VMs alone).
- Only `/dev/kvm` must be passed through. On the host, it's typically
  `crw-rw-rw-` (world-readable/writable), so any uid that can `open()` it
  gets KVM acceleration.
- `virsh -c qemu:///session domcapabilities` reports
  `<domain>kvm</domain>` once the daemons are running — empirically
  verified (2026-04-19) on `selkies-desktop-ov` at uid 1000.

## Tests baked into the layer

**Build-scope (run by `ov eval image <image>` without deploying):**

- `virtqemud-package` / `virtnetworkd-package` — package-existence probes using `package:` + `package_map:` (see below).
- `virsh-binary` — `/usr/bin/virsh` exists.
- `qemu-system-x86-binary` — `/usr/bin/qemu-system-x86_64` exists.
- `qemu-img-binary` — `/usr/bin/qemu-img` exists.

### Why package-existence, not binary-path, for the virtqemud/virtnetworkd tests

Debian/Ubuntu and Fedora/Arch bundle libvirt drivers differently. On Fedora/Arch the `libvirt-daemon-driver-qemu` / `libvirt-daemon-driver-network` subpackages ship `/usr/sbin/virtqemud` and `/usr/sbin/virtnetworkd` as standalone binaries. On Debian/Ubuntu those drivers are bundled inside `libvirt-daemon-system`, and the monolithic `libvirtd` binary replaces the per-driver split daemons — so a `file: /usr/sbin/virtqemud` probe hard-fails on deb even when the functionality is present.

The fix — introduced 2026-04 during Phase E — probes package presence with distro-specific names:

```yaml
- id: virtqemud-package
  package: libvirt-daemon-driver-qemu     # rpm name on Fedora
  package_map:
    archlinux: libvirt                    # Arch bundles into 'libvirt' metapackage
    debian: libvirt-daemon-system         # deb bundles all drivers here
    ubuntu: libvirt-daemon-system
  installed: true
```

See `/ov-eval:eval` "`package:` + `package_map:` pattern" for the resolution order (tag > base name > fallback).

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

## Cross-distro coverage

`rpm:` (Fedora), `pac:` (Arch), `deb:` (Debian generic + `ubuntu:24.04:` tag section). Debian/Ubuntu use `libvirt-daemon-system`, `libvirt-clients`, `qemu-system-x86`, `qemu-utils`, `qemu-kvm`, `virtinst`. The per-driver daemons (`virtqemud`, `virtnetworkd`) are bundled into `libvirt-daemon-system` on deb — supervisord still supervises libvirtd via the same fragment.

Drops on deb: `gvisor-tap-vsock`, `podman-machine` (not packaged; VM-mode networking features degrade gracefully on debian-coder / ubuntu-coder).

## Used In Images

- `/ov-selkies:selkies-desktop-ov` — rootless VM host inside a streaming desktop
- `/ov-distros:fedora-ov` — root VM host (same daemons, uid 0)
- `/ov-coder:arch-ov` — Arch counterpart
- `/ov-coder:debian-coder`, `/ov-coder:ubuntu-coder` — deb-based consumers (via `ov-full`)
- `/ov-distros:githubrunner` — VMs for CI workloads
- `/ov-distros:aurora`, `/ov-distros:bazzite-ai` — bootc siblings (currently `enabled: false`)

## Related Layers

- `/ov-coder:ov-full` — composition that pulls this layer into ov-toolchain images
- `/ov-distros:container-nesting` — pairs with this layer for images that need both nested containers AND nested VMs; also donates `/dev/kvm`-adjacent devices
- `/ov-infrastructure:socat` — part of `ov-full` alongside virtualization; used for VM console/hostfwd relays
- `/ov-infrastructure:gocryptfs` — part of `ov-full`; for encrypting VM disk storage

## Related Commands

- `/ov-vm:vm` — VM lifecycle (build, create, start, stop, ssh, console, destroy); defaults to `qemu:///session` at ov/vm.go:22
- `/ov-build:generate` — Containerfile generation; emits the supervisord `NN-virtualization.conf` fragment via `fragment_assembly` init model
- `/ov-image:layer` — layer authoring reference (tasks, vars, service blocks, tests syntax)
- `/ov-eval:eval` — declarative testing framework for the layer's `eval:` block (file, service, command verbs)

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
  2026-04-19 refactor added the supervisord programs + `requires:
  supervisord` + the deploy-scope tests, turning it from a
  package-only layer into a service provider.
