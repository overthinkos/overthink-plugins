---
name: vm
description: |
  MUST be invoked before any work involving: virtual machines, ov vm commands,
  kind:vm entities in vm.yml, cloud_image vs bootc source types,
  libvirt/QEMU backends, BIOS vs UEFI firmware, virtio-gpu video, or VM lifecycle.
---

# VM — Virtual Machine Management

## Disposability + deploy cross-ref

- **Disposability is a deploy property only.** A `kind: vm` entity carries no `disposable:` / `lifecycle:` field. Put `disposable: true` on the matching `deploy.<name>-vm` entry; that's what `ov update <vm-name>` checks. The `vm:` entity entry only describes VM shape (disk, RAM, SSH, cloud-init, libvirt), never authorization.
- **Deploy-level cross-ref**: a deployment with `target: vm` references its VM entity via `vm: <entity-name>`.
- **`ov update <vm-entity-name>`** searches `deploy:` for any entry with `target: vm` + `vm: <entity>`; disposable iff ANY such entry carries `disposable: true`. Absence of a matching disposable deploy entry → rebuild refused.

## Overview

`ov vm` commands build disk images and manage virtual machines via libvirt (default) or direct QEMU. VMs are declared as **`kind: vm` entities in `vm.yml`** — a first-class primitive alongside `kind: image` entries. Two source types:

- **`source.kind: cloud_image`** — fetches a pre-built qcow2 from an external URL (Arch, Fedora, Ubuntu, Debian, CentOS Cloud images). Renders a NoCloud seed ISO with cloud-init. Canonical example: `/ov-vm:arch`.
- **`source.kind: bootc`** — pairs with a `kind: image` entry that has `bootc: true`. Runs `bootc install to-disk` inside a privileged container. Canonical example: `/ov-vm:bazzite-bootc`.

VMs are not configured on kind:image entries — `image.vm:` / `image.libvirt:` are rejected at load time. `bootc: true` stays on a kind:image entry to mark it bootable. Legacy projects convert in one shot with `ov migrate` (see `/ov-build:migrate`). For the YAML authoring reference, see `/ov-vm:vms-catalog`; for the Go types, see `/ov-internals:vm-spec`.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Build QCOW2 | `ov vm build <name>` | Build disk image (`<name>` = kind:vm entity key) |
| Build RAW | `ov vm build <name> --type raw` | Build RAW disk |
| Create VM | `ov vm create <name>` | Provision libvirt/QEMU VM from the built disk |
| Start VM | `ov vm start <name>` | Start a stopped VM |
| Stop VM | `ov vm stop <name>` | Graceful shutdown |
| Force stop | `ov vm stop <name> --force` | Force stop (destroy) |
| Destroy VM | `ov vm destroy <name>` | Remove VM definition |
| Destroy + disk | `ov vm destroy <name> --disk` | Remove VM and delete disk |
| List VMs | `ov vm list [-a]` | List running (or all) VMs |
| Console | `ov vm console <name>` | Attach to serial console |
| SSH | `ov vm ssh <name>` | SSH into VM |
| GPU passthrough readiness | `ov vm gpu status` | Host IOMMU + vfio-pci + per-GPU group/driver report |
| GPU passthrough block | `ov vm gpu list` | Emit a ready-to-paste `libvirt.devices.hostdevs:` block (whole IOMMU group, `managed: "yes"`) |
| Load image into guest | `ov vm cp-image <vm> <ref> [--as <tag>]` | `podman save | scp | podman load` a host image into a running guest (idempotent) |

VM name convention: `ov-<name>[-<instance>]`. Default libvirt URI: `qemu:///session`.

## GPU passthrough (VFIO)

To pass a physical GPU through to a VM and (e.g.) run a CUDA container inside it:

1. **Host readiness** — `ov vm gpu status` confirms IOMMU is on (`intel_iommu=on`
   / `amd_iommu=on iommu=pt` on the kernel cmdline) and shows each GPU's IOMMU
   group + current driver. `ov doctor` reports the same under "VFIO / GPU
   passthrough".
2. **Author the hostdev** — `ov vm gpu list` prints a `libvirt.devices.hostdevs:`
   block covering **every** function in the GPU's IOMMU group (the GPU + its
   audio function + any bridge sibling — they must pass through together), each
   `managed: "yes"` (libvirt auto-binds them to vfio-pci on VM start and rebinds
   the host driver on stop). Paste it under the `kind: vm` entity. Use
   `firmware: uefi-insecure` and `backend: libvirt` (the qemu backend does not
   render `<hostdev>`). A PCI address is host-specific — do not commit it to a
   shared repo; keep it in a local/uncommitted entity or overlay.
3. **In-guest driver** — a passthrough guest needs the actual kernel module, not
   just the userspace container toolkit. Apply a kernel-driver layer (it
   blacklists the in-tree driver, regenerates the initramfs, and declares
   `reboot: true` so `ov deploy add vm:<name>` reboots the guest and waits for it
   to return). The `nvidia`-toolkit layer + `nvidia-ctk cdi generate` then makes
   the GPU available to containers via CDI (`--device nvidia.com/gpu=all`).

Code-43 workarounds for consumer NVIDIA cards are first-class libvirt fields:
`libvirt.features.kvm.hidden: on` and `libvirt.features.hyperv.vendor_id`.
Optional per-hostdev `rom: {bar: off}` / `driver: {name: vfio}` are supported.
Worked end-to-end example: the CachyOS `eval-cachyos-coder-vm` bed (see
`/ov-vm:cachyos`, `/ov-eval:eval`).

## Building Disk Images

```bash
ov vm build arch          # cloud_image: fetch qcow2, resize, render seed ISO
ov vm build bazzite-bootc --type qcow2   # bootc: bootc install to-disk
ov vm build <name> --size 40G        # disk_size override (CLI wins over vm.yml)
ov vm build <name> --root-size 10G   # bootc only: cap root partition
ov vm build <name> --console         # enable console output for debugging
ov vm build <name> --transport containers-storage   # bootc: local podman storage
```

Output: `output/<type>/disk.<type>`. For cloud_image, also `output/<type>/seed.iso`. ISO format is not supported — use qcow2 or raw.

The `--transport` flag (bootc only) controls how the container image is read: `registry` (pull from registry), `containers-storage` (local podman), `oci` (OCI dir), `oci-archive` (OCI tarball).

Source: `ov/vm_build.go`, `ov/vm_cloud_image.go`.

## VM Lifecycle

```bash
ov vm create arch                        # default resources from vm.yml
ov vm create <name> --ram 8G --cpus 4               # CLI overrides
ov vm create <name> --ssh-key auto                  # auto-detect SSH key (default)
ov vm create <name> --ssh-key generate              # generate new keypair
ov vm create <name> --ssh-key none                  # no key injection
ov vm create <name> --ssh-key ~/.ssh/id_ed25519.pub # specific key
ov vm create <name> -i prod                         # named instance
ov vm start <name>
ov vm stop <name>                                   # graceful
ov vm stop <name> --force                           # destroy
ov vm destroy <name>                                # remove VM definition
ov vm destroy <name> --disk                         # also delete disk image
ov vm list [-a]
ov vm console <name>                                # serial console
ov vm ssh <name>                                    # SSH with resolved user/port from vm.yml
ov vm ssh <name> -p 2224 -l arch                    # override user/port
ov vm ssh <name> -- ls /tmp                         # run command via SSH
```

**`ov vm create` regenerates the cloud-init seed ISO** (cloud_image sources
only). Edits to `vm.yml`'s `cloud_init:` block — runcmd entries, packages,
ov_install strategy, network-config — take effect on the next `ov vm create`
without requiring a full `ov vm build`. The qcow2 disk is left alone; only
the seed ISO is rewritten (via `RegenerateSeedISO` in `ov/vm_cloud_image.go`).
Use `ov vm destroy --disk` + `ov vm build` when you need a truly fresh disk
(e.g. when a previous failed cloud-init left stale state in `/home/<user>/`).

**`autostart: true`** on the entity makes `ov vm create` set libvirt's domain
autostart flag AND wire the session-boot trigger (`loginctl enable-linger` + a
generated per-VM user unit `ov-autostart-<domain>.service` that `virsh start`s
the domain at boot), so the VM starts at host boot. Requires the libvirt backend.
**`disk_size: 1T`** (or larger) is cheap — the qcow2 is sparse, so the virtual
size is just a ceiling and physical bytes grow on demand. A host directory can be
shared into the guest via `libvirt.devices.filesystems` (virtiofs) and mounted by
the `/ov-distros:workspace-mount` layer. See `/ov-vm:vms-catalog`.

## `kind: vm` entity reference

Canonical `vm.yml` shape, condensed from `/ov-vm:vms-catalog`:

```yaml
vms:
  # cloud_image source
  arch:
    source:
      kind: cloud_image
      url: https://fastly.mirror.pkgbuild.com/images/latest/Arch-Linux-x86_64-cloudimg.qcow2
      checksum: {type: sha256}                      # value auto-resolves from <url>.SHA256 sidecar
      base_user: arch                               # adopt pattern (no useradd)
    disk_size: 40G
    ram: 8G
    cpus: 4
    machine: q35
    firmware: bios                                  # BIOS preferred for cloud images — see decision matrix below
    network: {mode: user}
    ssh: {port: 2224, key_source: generate}
    cloud_init:
      packages: [sudo, spice-vdagent]
      ov_install: {strategy: auto}
    libvirt:
      devices:
        video: [{model: virtio, vram: 65536, heads: 1, accel3d: false}]
        graphics: [{type: spice, autoport: "yes", listen: 127.0.0.1}]

  # bootc source
  bazzite-bootc:
    source:
      kind: bootc
      image: bazzite                                # kind:image entry
    disk_size: 80 GiB
    ram: 16G
    cpus: 6
    ssh: {user: root, port: 2250}
```

Every field in `VmSpec` applies to both source kinds (parity guarantee). See `/ov-internals:vm-spec` for the Go type reference and `/ov-vm:vms-catalog` for the full field-by-field authoring guide.

## Backend matrix

Resolution order: **libvirt → qemu** (auto-detected). Override via `ov settings set vm.backend libvirt`.

| Backend | Requires | When to pick |
|---|---|---|
| `libvirt` (default) | libvirt session daemon socket | Almost always — full domain XML, portForward with passt, SPICE console, per-VM NVRAM |
| `qemu` | `qemu-system-*` binary | Direct QEMU; air-gapped hosts where libvirt isn't installed |

**Socket path probing for libvirt ≥ 8.0**: modular libvirt splits into `virtqemud` / `virtnetworkd` / ... — the socket is `virtqemud-sock`, not `libvirt-sock`. `libvirtSessionSocket()` probes `virtqemud-sock` first, falls back to `libvirt-sock` on older setups. Symptom of a misprobe: `ConnectToURI(QEMUSession)` returns `End of file while reading data: Input/output error` — means the daemon isn't accepting on the socket you reached.

### Prereq: libvirt user-session daemon

`backend: libvirt` requires `virtqemud.service` (or older monolithic
`libvirtd.service`) running as a **user**-service — no sudo needed:

```bash
systemctl --user enable --now virtqemud.service
```

`ov vm create` best-effort starts `virtqemud.service` (and falls back
to `libvirtd.service`) before backend resolution, so on hosts where
the unit is installed-but-not-started, the first VM create works
without manual intervention. If the unit isn't installed at all
(libvirt missing entirely), `resolveVmBackend()` surfaces the
explicit error `"libvirt backend requires libvirt session daemon"`
with the remediation hint above.

For projects whose eval beds use `ov eval libvirt …` and `ov eval
spice …` probes (e.g., the project's `arch:` VM template), pin the
backend explicitly via `backend: libvirt` on the kind:vm entity —
`backend: auto` would silently fall through to qemu when the daemon
is missing, breaking every libvirt-RPC probe with a confusing
"no such file or directory" error 5+ minutes into the eval-live
timeout. `arch:` and `k3s-vm:` carry this pin (see `/ov-eval:eval`
"kind: eval beds").

Source: `ov/vm.go` (`resolveVmBackend`, `startLibvirtUserSession`),
`ov/vm_libvirt.go`, `ov/vm_qemu.go`.

## BIOS vs UEFI decision matrix

The `firmware:` field controls the VM boot path. Choice matters more than most authors realize.

| Firmware | When to pick |
|---|---|
| `bios` (default) | Cloud images (Arch, Fedora Cloud, Ubuntu Cloud, Debian Cloud, CentOS Cloud) that ship a GPT BIOS boot partition. Avoids stale BOOTX64.EFI issues. No OVMF package dependency. No per-VM NVRAM. See `/ov-vm:arch` Finding B for the RCA. |
| `uefi-insecure` | bootc Fedora images (ship signed BOOTX64.EFI built at image-build time). Guests needing GPT > 2 TiB disks. ARM hosts (aarch64). |
| `uefi-secure` | Guests needing Secure Boot (signed kernel + initramfs). Locks to Microsoft UEFI CA keys. |

See `/ov-internals:ovmf` for the per-distro OVMF_CODE/OVMF_VARS path table and the per-VM NVRAM mechanics. When `firmware: bios`, `ResolveOvmfForSpec` returns empty strings and the libvirt renderer skips `<loader>`/`<nvram>` entirely.

## Video-model choice

`libvirt.devices.video[0].model` drives the VM's paravirtualized GPU.

| Model | Default for | Why |
|---|---|---|
| `virtio` (virtio-gpu) | **Linux guests — modern default** | Native `virtio_gpu` kernel DRM driver (4.16+), Wayland-native, simpler config, no X-server-specific driver needed |
| `qxl` | Legacy X11 SPICE use | 2010-era Red Hat SPICE stack; simpledrm→qxldrmfb takeover race under UEFI (see `/ov-vm:arch` Finding B); more knobs (ram_size + vram_size + vram64_size_mb + vgamem_mb) |
| `cirrus` | BIOS fallback only | Low resolution, no acceleration |
| `none` | Headless VMs | No framebuffer at all |

SPICE is video-model-agnostic — it streams pixels from virtio-gpu as well as from QXL. See `/ov-internals:libvirt-renderer` "video-model choice" for the full decision rationale.

## SSH Key Injection

SSH keys inject via two additive channels (both on by default for cloud_image VMs):

- **SMBIOS type 11 OEM strings** — systemd-ssh-generator (systemd ≥ v250) materializes the pubkey into `~<user>/.ssh/authorized_keys` during early boot. Works even without cloud-init.
- **cloud-init user-data** — `users:[...].ssh_authorized_keys` list in the rendered user-data on the NoCloud seed ISO.

The `--ssh-key` flag (and `vms.<name>.ssh.key_source`) controls the source of the pubkey:

| Value | Behavior |
|-------|----------|
| `auto` (default) | Uses first `~/.ssh/*.pub` found |
| `generate` | Creates new keypair in `~/.local/share/ov/vm/ov-<name>/id_ed25519`. Idempotent across rebuilds. |
| `none` | No key injection |
| `/path/to/key.pub` | Uses the specified public key |

See `/ov-internals:vm-spec` for the `VmSSH.KeyInjection` tri-state (`auto` | `enabled` | `disabled`) per channel, and `/ov-internals:cloud-init-renderer` for the `composeUsers` adopt-merge pattern that delivers the key into the guest.

## User-Level Defaults

Set via `ov settings set`:

| Key | Env Var | Default |
|-----|---------|---------|
| `vm.backend` | `OV_VM_BACKEND` | `auto` |

Per-VM overrides live in `vm.yml`. The user-level defaults exist only for fields that don't have a per-VM equivalent (backend is the main one). For anything else, declare it in `vms.<name>`.

## Libvirt XML configuration

Primary surface is structured `LibvirtConfig` in `vms.<name>.libvirt:` (features, CPU, clock, devices, sysinfo, launch_security, etc.). See `/ov-internals:libvirt-renderer` for the full schema.

Raw-XML escape hatch: `vms.<name>.libvirt.snippets:` (list of strings) — classified by element name. Device-scoped elements go into `<devices>`, domain-scoped before `</domain>`. Deduplicated by exact string match.

Layer-level raw snippets: `layer.yml` `libvirt.snippets:` is supported for layers that contribute device XML (e.g., `/ov-distros:qemu-guest-agent` contributes the virtio-serial channel). Image-level `libvirt: [...]` is not a valid field — VM XML lives on the `kind: vm` entity.

Source: `ov/libvirt.go`, `ov/libvirt_render.go`, `ov/libvirt_render_devices.go`.

## Common Workflows

### Build and run a cloud_image VM

```bash
ov vm build arch              # fetches qcow2, resizes, renders seed ISO
ov vm create arch
ssh -p 2224 -i ~/.local/share/ov/vm/ov-arch/id_ed25519 arch@127.0.0.1
```

### Build and run a bootc VM

```bash
ov image build bazzite             # container image must exist first
ov vm build bazzite-bootc
ov vm create bazzite-bootc
ov vm start bazzite-bootc
ov vm ssh bazzite-bootc
```

### Apply layers inside a VM

```bash
ov vm create arch
ov deploy add vm:arch ripgrep         # apply ripgrep layer over SSH
ov deploy add vm:arch fedora-coder --add-layer team-extras
ov deploy del vm:arch                 # reverse all applied layers
```

See `/ov-core:deploy` "vm: target" for the `ov deploy add vm:<name>` surface and `/ov-internals:vm-deploy-target` for the executor model.

### Debug VM boot issues

```bash
ov vm build <name> --console     # enable console output during disk build
ov vm create <name>
ov vm start <name>
ov vm console <name>             # watch boot messages
```

### Multiple VM instances

```bash
ov vm create <name> -i dev
ov vm create <name> -i staging
ov vm start <name> -i dev
ov vm ssh <name> -i dev
```

## Known live-tested caveats

Non-obvious issues that surface only when VMs actually boot. `/ov-vm:bazzite-bootc` is the canonical end-to-end bootc worked example; `/ov-vm:arch` is the canonical cloud_image worked example.

### Privileged container needs `-v /dev:/dev` (bootc)

`ov vm build` runs `bootc install to-disk --via-loopback` inside a privileged container. `--privileged` alone is not enough — the container's `/dev` is a tmpfs in its own mount namespace, so `losetup`'s `LOOP_CTL_GET_FREE` creates a `/dev/loopN` in the kernel but the node is invisible inside the container. `ov/vm_build.go` adds `-v /dev:/dev` so the container shares the host's `/dev`.

### virtqemud socket probing (libvirt ≥ 8.0)

Modular libvirt daemons: the session socket moved from `libvirt-sock` to `virtqemud-sock`. Probe order in `libvirtSessionSocket()`: virtqemud-sock first, libvirt-sock fallback. `ConnectToURI(QEMUSession)` uses `qemu:///session`, not `qemu:///system`.

### `<backend type='passt'/>` required for `<portForward>`

libvirt ≥ 9.x `<portForward>` on `<interface type='user'>` only activates with `<backend type='passt'/>`. Emitted automatically by `renderDefaultInterface`; if you author a custom interface, include the backend line. Symptom when missing: ssh on port 2224 accepts TCP but lands in the wrong guest port. See `/ov-internals:libvirt-renderer`.

### Per-VM NVRAM required for UEFI

When `firmware: uefi-insecure` or `firmware: uefi-secure`, each VM gets its own NVRAM file under `~/.local/share/ov/vm/ov-<name>/nvram.fd`, copied from the OVMF_VARS template. Preserves UEFI variable state across reboots. See `/ov-internals:ovmf`.

### Resource sizing over package pruning (cloud_image)

Cloud-init `packages: [spice-vdagent]` pulls GTK3 + X11 (~200 MB download, ~1 GB installed). Running at 2 GiB RAM stalls cloud-init. **Check host inventory (`free -h`, `nproc`) before sizing** — give the VM what it needs (8 GiB + 4 cpus for a desktop-class workload is reasonable on a 16-core / 61 GiB host). See `/ov-vm:arch` Finding C for the live-test bisect.

### Rootful ↔ rootless podman storage split (bootc)

Under `engine.rootful=sudo`, `ov vm build` invokes `sudo podman` which reads from `/var/lib/containers/storage`, not your rootless storage. After every `ov image build`, refresh:

```bash
podman save ghcr.io/<registry>/<image>:latest -o /tmp/img.tar
sudo podman load -i /tmp/img.tar && rm -f /tmp/img.tar
```

Symptom without refresh: `Trying to pull ... 403 Forbidden`.

### `engine.rootful=machine` path (rootless containers)

When `ov vm build` runs from inside a rootless container (no host `sudo`), the engine auto-picks `engine.rootful=machine` — spawns a podman-machine VM with its own rootful storage (`podman --connection ov-root`). Cross-storage load pattern:

```bash
podman save -o /tmp/bootc.tar ghcr.io/<registry>/<image>:latest
podman --connection ov-root load -i /tmp/bootc.tar
rm /tmp/bootc.tar
ov vm build <name> --transport containers-storage
```

`--transport containers-storage` forces `bootc install` to pull from the machine's local store. Worked example: `/ov-openclaw:openclaw-desktop` "Two-level nested-virtualization proof".

### `ov vm stop` state-sync race under QEMU backend

Under the QEMU backend, `ov vm stop <name>` returns success + exit 0, but `ov vm list -a` still reports the VM as `running` until `ov vm destroy` runs. Libvirt backend unaffected. Workaround: proceed to `ov vm destroy` rather than treating residual "running" as a stop failure.

### Host kernel/module-dir mismatch

Rolling distros after a kernel upgrade: `/lib/modules/$(uname -r)` missing, `modprobe loop` fails, VM disk build hangs. **Resolution: reboot.** Check: `ls /lib/modules/$(uname -r) >/dev/null 2>&1 && echo ok || echo "reboot needed"`.

### Disk exhaustion from `output/`

`bootc install to-disk` writes both a ~40 GiB sparse raw AND a qcow2. Failed `qemu-img convert` leaves both behind. Clean `output/raw/disk.raw` after successful qcow2 conversion; watch `df -h /` before fresh VM builds.

### `qemu-guest-agent` inactive under QEMU user-net

Expected. The agent needs a `virtio-serial` channel that ov's QEMU backend doesn't expose under user-net. Switch to libvirt backend (`ov settings set vm.backend libvirt`) to activate it. See `/ov-distros:qemu-guest-agent`.

## Cross-References

- `/ov-vm:vms-catalog` — **vm.yml authoring reference** (VmSpec schema, source.kind, adopt pattern)
- `/ov-vm:arch` — canonical cloud_image VM (BIOS / virtio-gpu / resource sizing / stale BOOTX64.EFI RCA)
- `/ov-vm:aurora-bootc`, `/ov-vm:bazzite-bootc` — bootc VMs
- `/ov-internals:vm-spec` — Go type reference; validation rules; migration map
- `/ov-internals:libvirt-renderer` — RenderDomain + device emission + passt backend + virtio-gpu
- `/ov-internals:cloud-init-renderer` — RenderCloudInit + composeUsers + seed ISO + ov_install
- `/ov-internals:vm-deploy-target` — VmDeployTarget / SSHExecutor / VmDeployState
- `/ov-internals:ovmf` — UEFI firmware path resolution; per-VM NVRAM; bios-skips-loader sentinel
- `/ov-internals:cutover-policy` — hard cutover policy
- `/ov-build:migrate` — `ov migrate` legacy conversion
- `/ov-core:deploy` — `ov deploy add vm:<name> <ref>` in-guest layer application
- `/ov-build:pull` — fetch container images into local storage (prereq for bootc VM builds)
- `/ov-build:build` — building container images before VM disk builds
- `/ov-image:layer` — `libvirt.snippets:` field in layer.yml
- `/ov-openclaw:openclaw-desktop` — two-level nested-virtualization proof
- `/ov-distros:bootc-base` — sshd + qemu-guest-agent + bootc-config bundle
- `/ov-distros:bootc-config` — bootc-side boot wiring (tty1 autologin, graphical target, systemd-user supervisord)
- `/ov-distros:cloud-init` — guest-side cloud-init package (complements host-side seed ISO rendering)
- `/ov-distros:qemu-guest-agent` — virtio-serial channel

## When to Use This Skill

**MUST be invoked** when the task involves virtual machines, ov vm commands, kind:vm entities, cloud_image vs bootc source types, libvirt/QEMU backends, BIOS vs UEFI firmware choice, or VM lifecycle management. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Standalone workflow. VM management is separate from container lifecycle, but `ov deploy add vm:<name>` bridges into the shared InstallPlan + DeployTarget machinery.

## Live-deploy verification is mandatory (see `/ov-eval:eval` 10 standards)

Changes that touch this verb's output must reach a healthy deployment on a target explicitly marked `disposable: true` (see `/ov-internals:disposable`). Use `ov update <name>` to destroy + rebuild unattended on any disposable target. Never experiment on a non-disposable deploy — set up a disposable one first with `ov deploy add <name> <ref> --disposable` or mark a VM in vm.yml.

**After committing the source-level fix, `ov update` the disposable target ONCE MORE from clean and re-run the full verification.** A fix that passes only on a hand-patched target is not a real fix — it's a regression waiting for the next unrelated rebuild. Paste BOTH the exploratory-pass output and the fresh-rebuild-pass output into the conversation.

Unit tests + a clean compile are necessary but not sufficient. See CLAUDE.md R1–R10.
