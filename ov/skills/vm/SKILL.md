---
name: vm
description: |
  MUST be invoked before any work involving: virtual machines, ov vm commands,
  kind:vm entities in vms.yml, cloud_image vs bootc source types,
  libvirt/QEMU backends, BIOS vs UEFI firmware, virtio-gpu video, or VM lifecycle.
---

# VM — Virtual Machine Management

## Overview

`ov vm` commands build disk images and manage virtual machines via libvirt (default) or direct QEMU. VMs are declared as **`kind: vm` entities in `vms.yml`** — a first-class primitive alongside `kind: image` entries. Two source types:

- **`source.kind: cloud_image`** — fetches a pre-built qcow2 from an external URL (Arch, Fedora, Ubuntu, Debian, CentOS Cloud images). Renders a NoCloud seed ISO with cloud-init. Canonical example: `/ov-vms:arch-cloud-base`.
- **`source.kind: bootc`** — pairs with a `kind: image` entry that has `bootc: true`. Runs `bootc install to-disk` inside a privileged container. Canonical example: `/ov-vms:selkies-desktop-bootc-bootc`.

The legacy `image.bootc: true` + `image.vm: {...}` + `image.libvirt: [...]` fields were **deleted in the hard cutover**. Legacy projects convert in one shot with `ov migrate vm-spec` (see `/ov:migrate`). For the YAML authoring reference, see `/ov-vms:vms`; for the Go types, see `/ov-dev:vm-spec`.

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

VM name convention: `ov-<name>[-<instance>]`. Default libvirt URI: `qemu:///session`.

## Building Disk Images

```bash
ov vm build arch-cloud-base          # cloud_image: fetch qcow2, resize, render seed ISO
ov vm build selkies-desktop-bootc-bootc --type qcow2   # bootc: bootc install to-disk
ov vm build <name> --size 40G        # disk_size override (CLI wins over vms.yml)
ov vm build <name> --root-size 10G   # bootc only: cap root partition
ov vm build <name> --console         # enable console output for debugging
ov vm build <name> --transport containers-storage   # bootc: local podman storage
```

Output: `output/<type>/disk.<type>`. For cloud_image, also `output/<type>/seed.iso`. ISO format is not supported — use qcow2 or raw.

The `--transport` flag (bootc only) controls how the container image is read: `registry` (pull from registry), `containers-storage` (local podman), `oci` (OCI dir), `oci-archive` (OCI tarball).

Source: `ov/vm_build.go`, `ov/vm_cloud_image.go`.

## VM Lifecycle

```bash
ov vm create arch-cloud-base                        # default resources from vms.yml
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
ov vm ssh <name>                                    # SSH with resolved user/port from vms.yml
ov vm ssh <name> -p 2224 -l arch                    # override user/port
ov vm ssh <name> -- ls /tmp                         # run command via SSH
```

**`ov vm create` regenerates the cloud-init seed ISO** (cloud_image sources
only). Edits to `vms.yml`'s `cloud_init:` block — runcmd entries, packages,
ov_install strategy, network-config — take effect on the next `ov vm create`
without requiring a full `ov vm build`. The qcow2 disk is left alone; only
the seed ISO is rewritten (via `RegenerateSeedISO` in `ov/vm_cloud_image.go`).
Use `ov vm destroy --disk` + `ov vm build` when you need a truly fresh disk
(e.g. when a previous failed cloud-init left stale state in `/home/<user>/`).

## `kind: vm` entity reference

Canonical `vms.yml` shape, condensed from `/ov-vms:vms`:

```yaml
vms:
  # cloud_image source
  arch-cloud-base:
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
  selkies-desktop-bootc-bootc:
    source:
      kind: bootc
      image: selkies-desktop-bootc                  # kind:image entry
    disk_size: 40 GiB
    ram: 8G
    cpus: 4
    ssh: {user: root, port: 2250}
```

Every field in `VmSpec` applies to both source kinds (parity guarantee). See `/ov-dev:vm-spec` for the Go type reference and `/ov-vms:vms` for the full field-by-field authoring guide.

## Backend matrix

Resolution order: **libvirt → qemu** (auto-detected). Override via `ov settings set vm.backend libvirt`.

| Backend | Requires | When to pick |
|---|---|---|
| `libvirt` (default) | libvirt session daemon socket | Almost always — full domain XML, portForward with passt, SPICE console, per-VM NVRAM |
| `qemu` | `qemu-system-*` binary | Direct QEMU; air-gapped hosts where libvirt isn't installed |

**Socket path probing for libvirt ≥ 8.0**: modular libvirt splits into `virtqemud` / `virtnetworkd` / ... — the socket is `virtqemud-sock`, not `libvirt-sock`. `libvirtSessionSocket()` probes `virtqemud-sock` first, falls back to `libvirt-sock` on older setups. Symptom of a misprobe: `ConnectToURI(QEMUSession)` returns `End of file while reading data: Input/output error` — means the daemon isn't accepting on the socket you reached.

Source: `ov/vm.go`, `ov/vm_libvirt.go`, `ov/vm_qemu.go`.

## BIOS vs UEFI decision matrix

The `firmware:` field controls the VM boot path. Choice matters more than most authors realize.

| Firmware | When to pick |
|---|---|
| `bios` (default) | Cloud images (Arch, Fedora Cloud, Ubuntu Cloud, Debian Cloud, CentOS Cloud) that ship a GPT BIOS boot partition. Avoids stale BOOTX64.EFI issues. No OVMF package dependency. No per-VM NVRAM. See `/ov-vms:arch-cloud-base` Finding B for the RCA. |
| `uefi-insecure` | bootc Fedora images (ship signed BOOTX64.EFI built at image-build time). Guests needing GPT > 2 TiB disks. ARM hosts (aarch64). |
| `uefi-secure` | Guests needing Secure Boot (signed kernel + initramfs). Locks to Microsoft UEFI CA keys. |

See `/ov-dev:ovmf` for the per-distro OVMF_CODE/OVMF_VARS path table and the per-VM NVRAM mechanics. When `firmware: bios`, `ResolveOvmfForSpec` returns empty strings and the libvirt renderer skips `<loader>`/`<nvram>` entirely.

## Video-model choice

`libvirt.devices.video[0].model` drives the VM's paravirtualized GPU.

| Model | Default for | Why |
|---|---|---|
| `virtio` (virtio-gpu) | **Linux guests — modern default** | Native `virtio_gpu` kernel DRM driver (4.16+), Wayland-native, simpler config, no X-server-specific driver needed |
| `qxl` | Legacy X11 SPICE use | 2010-era Red Hat SPICE stack; simpledrm→qxldrmfb takeover race under UEFI (see `/ov-vms:arch-cloud-base` Finding B); more knobs (ram_size + vram_size + vram64_size_mb + vgamem_mb) |
| `cirrus` | BIOS fallback only | Low resolution, no acceleration |
| `none` | Headless VMs | No framebuffer at all |

SPICE is video-model-agnostic — it streams pixels from virtio-gpu as well as from QXL. See `/ov-dev:libvirt-renderer` "video-model choice" for the full decision rationale.

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

See `/ov-dev:vm-spec` for the `VmSSH.KeyInjection` tri-state (`auto` | `enabled` | `disabled`) per channel, and `/ov-dev:cloud-init-renderer` for the `composeUsers` adopt-merge pattern that delivers the key into the guest.

## User-Level Defaults

Set via `ov settings set`:

| Key | Env Var | Default |
|-----|---------|---------|
| `vm.backend` | `OV_VM_BACKEND` | `auto` |

Per-VM overrides live in `vms.yml`. The user-level defaults exist only for fields that don't have a per-VM equivalent (backend is the main one). For anything else, declare it in `vms.<name>`.

## Libvirt XML configuration

Primary surface is structured `LibvirtConfig` in `vms.<name>.libvirt:` (features, CPU, clock, devices, sysinfo, launch_security, etc.). See `/ov-dev:libvirt-renderer` for the full schema.

Raw-XML escape hatch: `vms.<name>.libvirt.snippets:` (list of strings) — classified by element name. Device-scoped elements go into `<devices>`, domain-scoped before `</domain>`. Deduplicated by exact string match.

Layer-level raw snippets: `layer.yml` `libvirt.snippets:` still supported for layers that contribute device XML (e.g., `/ov-layers:qemu-guest-agent` contributes the virtio-serial channel). Image-level `libvirt: [...]` was **removed** in the hard cutover — it had no home in the new VM model.

Source: `ov/libvirt.go`, `ov/libvirt_render.go`, `ov/libvirt_render_devices.go`.

## Common Workflows

### Build and run a cloud_image VM

```bash
ov vm build arch-cloud-base              # fetches qcow2, resizes, renders seed ISO
ov vm create arch-cloud-base
ssh -p 2224 -i ~/.local/share/ov/vm/ov-arch-cloud-base/id_ed25519 arch@127.0.0.1
```

### Build and run a bootc VM

```bash
ov image build selkies-desktop-bootc             # container image must exist first
ov vm build selkies-desktop-bootc-bootc
ov vm create selkies-desktop-bootc-bootc
ov vm start selkies-desktop-bootc-bootc
ov vm ssh selkies-desktop-bootc-bootc
```

### Apply layers inside a VM

```bash
ov vm create arch-cloud-base
ov deploy add vm:arch-cloud-base ripgrep         # apply ripgrep layer over SSH
ov deploy add vm:arch-cloud-base fedora-coder --add-layer team-extras
ov deploy del vm:arch-cloud-base                 # reverse all applied layers
```

See `/ov:deploy` "vm: target" for the `ov deploy add vm:<name>` surface and `/ov-dev:vm-deploy-target` for the executor model.

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

Non-obvious issues that surface only when VMs actually boot. `/ov-images:selkies-desktop-bootc` is the canonical end-to-end bootc worked example; `/ov-vms:arch-cloud-base` is the canonical cloud_image worked example.

### Privileged container needs `-v /dev:/dev` (bootc)

`ov vm build` runs `bootc install to-disk --via-loopback` inside a privileged container. `--privileged` alone is not enough — the container's `/dev` is a tmpfs in its own mount namespace, so `losetup`'s `LOOP_CTL_GET_FREE` creates a `/dev/loopN` in the kernel but the node is invisible inside the container. `ov/vm_build.go` adds `-v /dev:/dev` so the container shares the host's `/dev`.

### virtqemud socket probing (libvirt ≥ 8.0)

Modular libvirt daemons: the session socket moved from `libvirt-sock` to `virtqemud-sock`. Probe order in `libvirtSessionSocket()`: virtqemud-sock first, libvirt-sock fallback. `ConnectToURI(QEMUSession)` uses `qemu:///session`, not `qemu:///system`.

### `<backend type='passt'/>` required for `<portForward>`

libvirt ≥ 9.x `<portForward>` on `<interface type='user'>` only activates with `<backend type='passt'/>`. Emitted automatically by `renderDefaultInterface`; if you author a custom interface, include the backend line. Debugged live in April 2026: ssh on port 2224 accepted TCP but landed in the wrong guest port when the backend was missing. See `/ov-dev:libvirt-renderer`.

### Per-VM NVRAM required for UEFI

When `firmware: uefi-insecure` or `firmware: uefi-secure`, each VM gets its own NVRAM file under `~/.local/share/ov/vm/ov-<name>/nvram.fd`, copied from the OVMF_VARS template. Preserves UEFI variable state across reboots. See `/ov-dev:ovmf`.

### Resource sizing over package pruning (cloud_image)

Cloud-init `packages: [spice-vdagent]` pulls GTK3 + X11 (~200 MB download, ~1 GB installed). Running at 2 GiB RAM stalls cloud-init. **Check host inventory (`free -h`, `nproc`) before sizing** — give the VM what it needs (8 GiB + 4 cpus for a desktop-class workload is reasonable on a 16-core / 61 GiB host). See `/ov-vms:arch-cloud-base` Finding C for the live-test bisect.

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

`--transport containers-storage` forces `bootc install` to pull from the machine's local store. Worked example: `/ov-images:selkies-desktop-ov` "Two-level nested-virtualization proof".

### `ov vm stop` state-sync race under QEMU backend

Observed 2026-04-19: `ov vm stop <name>` returns success + exit 0, but `ov vm list -a` still reports the VM as `running` until `ov vm destroy` runs. Libvirt backend unaffected. Workaround: proceed to `ov vm destroy` rather than treating residual "running" as a stop failure.

### Host kernel/module-dir mismatch

Rolling distros after a kernel upgrade: `/lib/modules/$(uname -r)` missing, `modprobe loop` fails, VM disk build hangs. **Resolution: reboot.** Check: `ls /lib/modules/$(uname -r) >/dev/null 2>&1 && echo ok || echo "reboot needed"`.

### Disk exhaustion from `output/`

`bootc install to-disk` writes both a ~40 GiB sparse raw AND a qcow2. Failed `qemu-img convert` leaves both behind. Clean `output/raw/disk.raw` after successful qcow2 conversion; watch `df -h /` before fresh VM builds.

### `qemu-guest-agent` inactive under QEMU user-net

Expected. The agent needs a `virtio-serial` channel that ov's QEMU backend doesn't expose under user-net. Switch to libvirt backend (`ov settings set vm.backend libvirt`) to activate it. See `/ov-layers:qemu-guest-agent`.

## Cross-References

- `/ov-vms:vms` — **vms.yml authoring reference** (VmSpec schema, source.kind, adopt pattern)
- `/ov-vms:arch-cloud-base` — canonical cloud_image VM (BIOS / virtio-gpu / resource sizing / stale BOOTX64.EFI RCA)
- `/ov-vms:aurora-bootc`, `/ov-vms:bazzite-ai-bootc`, `/ov-vms:openclaw-browser-bootc-bootc`, `/ov-vms:selkies-desktop-bootc-bootc` — bootc VMs
- `/ov-dev:vm-spec` — Go type reference; validation rules; legacy migration map
- `/ov-dev:libvirt-renderer` — RenderDomain + device emission + passt backend + virtio-gpu
- `/ov-dev:cloud-init-renderer` — RenderCloudInit + composeUsers + seed ISO + ov_install
- `/ov-dev:vm-deploy-target` — VmDeployTarget / SSHExecutor / VmDeployState
- `/ov-dev:ovmf` — UEFI firmware path resolution; per-VM NVRAM; bios-skips-loader sentinel
- `/ov-dev:cutover-policy` — hard cutover policy — why the legacy surface was deleted
- `/ov:migrate` — `ov migrate vm-spec` legacy conversion
- `/ov:deploy` — `ov deploy add vm:<name> <ref>` in-guest layer application
- `/ov:pull` — fetch container images into local storage (prereq for bootc VM builds)
- `/ov:build` — building container images before VM disk builds
- `/ov:layer` — `libvirt.snippets:` field in layer.yml
- `/ov-images:selkies-desktop-bootc` — canonical end-to-end bootc worked example
- `/ov-images:selkies-desktop-ov` — two-level nested-virtualization proof
- `/ov-layers:bootc-base` — sshd + qemu-guest-agent + bootc-config bundle
- `/ov-layers:bootc-config` — bootc-side boot wiring (tty1 autologin, graphical target, systemd-user supervisord)
- `/ov-layers:cloud-init` — guest-side cloud-init package (complements host-side seed ISO rendering)
- `/ov-layers:qemu-guest-agent` — virtio-serial channel

## When to Use This Skill

**MUST be invoked** when the task involves virtual machines, ov vm commands, kind:vm entities, cloud_image vs bootc source types, libvirt/QEMU backends, BIOS vs UEFI firmware choice, or VM lifecycle management. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Standalone workflow. VM management is separate from container lifecycle, but `ov deploy add vm:<name>` bridges into the shared InstallPlan + DeployTarget machinery.

## Live-deploy verification is mandatory (see `/ov:test` 10 standards)

Changes that touch this verb's output must reach a healthy deployment on a target explicitly marked `disposable: true` (see `/ov-dev:disposable`). Use `ov rebuild <name>` to destroy + rebuild unattended on any disposable target. Never experiment on a non-disposable deploy — set up a disposable one first with `ov deploy add <name> <ref> --disposable` or mark a VM in vms.yml.

**After committing the source-level fix, `ov rebuild` the disposable target ONCE MORE from clean and re-run the full verification.** A fix that passes only on a hand-patched target is not a real fix — it's a regression waiting for the next unrelated rebuild. Paste BOTH the exploratory-pass output and the fresh-rebuild-pass output into the conversation.

Unit tests + a clean compile are necessary but not sufficient. See CLAUDE.md R1–R10.
