---
name: libvirt-renderer
description: |
  Pure renderer from VmSpec + LibvirtConfig to libvirt domain XML and QEMU argv.
  Covers RenderDomain, device emission (passt backend, portForward attribute
  order, virtio-gpu defaults), firmware plumbing, and LibvirtConfig schema shape.
  Source: charly/libvirt_schema.go, charly/libvirt_render.go, charly/libvirt_render_devices.go,
  charly/qemu_render.go. MUST be invoked before editing libvirt XML emission.
---

# libvirt-renderer

The libvirt renderer converts `VmSpec` + `LibvirtConfig` into a libvirt domain XML (`RenderDomain`) and — for the direct-QEMU backend — into a QEMU argv array (`RenderQemuArgv`). Pure functions: given the same inputs, they produce the same output; no side effects. Consumed by `charly vm create` (libvirt path) and `charly vm create --backend qemu` (direct path).

## Source files

| File | Contents | LOC |
|---|---|---|
| `charly/libvirt_schema.go` | `LibvirtConfig` + 30+ sub-types (features, CPU, clock, memory backing, memtune, numatune, cputune, devices, seclabel, launch security, resource, sysinfo) | ~470 |
| `charly/libvirt_render.go` | `RenderDomain` top-level composition; firmware plumbing (D17); SMBIOS credentials | ~800 |
| `charly/libvirt_render_devices.go` | `<devices>` child emitters — channels, graphics, video, rng, memballoon, hostdev, interface (with portForward), filesystem | ~700 |
| `charly/qemu_render.go` | `RenderQemuArgv` for direct-QEMU backend | ~340 |
| `charly/libvirt_validate.go` | `ValidateLibvirtConfig` |

## LibvirtConfig top-level

```go
type LibvirtConfig struct {
    Snippets []string                      // raw XML escape hatch, classified by isDeviceElement
    Features       *LibvirtFeatures        // ACPI/APIC/PAE/SMM/HAP/VMPort/PMU/HyperV/KVM
    CPU            *LibvirtCPU             // mode, model, features, topology, numa
    Clock          *LibvirtClock           // offset, timers
    MemoryBacking  *LibvirtMemoryBacking   // hugepages, nosharepages, locked, source
    MemTune        *LibvirtMemTune         // hard_limit, soft_limit, swap_hard_limit
    NUMATune       *LibvirtNUMATune        // memory, memnode
    CPUTune        *LibvirtCPUTune         // vcpupin, emulatorpin, iothreadpin, shares, period
    IOThreads      int
    Devices        *LibvirtDevices         // interfaces, disks, channels, graphics, video, rng, …
    SecLabel       *LibvirtSecLabel        // SELinux/AppArmor
    LaunchSecurity *LibvirtLaunchSecurity  // AMD SEV / Intel TDX
    Resource       *LibvirtResource        // partition
    SysInfo        *LibvirtSysInfo         // SMBIOS — type 11 OEM strings used for ssh-key injection
}
```

**Structured first; raw XML is a last resort.** `Snippets` exists for the rare case where libvirt gained a new element before charly's schema caught up. The legacy list-of-strings form `libvirt: ["<xml>", ...]` on `kind: box` entries was deleted in the cutover — raw XML now lives only in `vms.<name>.libvirt.snippets:` (new), on layer-level `libvirt.snippets:` fields, and the `InjectLibvirtXML` post-processor still handles both paths.

## Device-level gotchas learned in live testing

### `<backend type='passt'/>` required for `<portForward>`

libvirt ≥ 9.x `<portForward>` elements on `<interface type='user'>` only activate when the interface declares `<backend type='passt'/>`. Without it, libvirt silently drops the port mapping — guest is reachable internally but host has nothing on the forwarded port. Emitted automatically in `renderDefaultInterface`; if you author a custom interface in `libvirt.devices.interfaces[]`, include the backend line.

### `<portForward>` start/to attribute order

The element's attributes are **start=HOST port, to=GUEST port** (not the other way around). Reversing them maps incoming guest traffic to the host, which is exactly wrong. The renderer enforces this ordering; unit tests in `libvirt_test.go` guard against regression. The live-test bisect that caught this: ssh on port 2224 connected but landed in the wrong port inside the guest.

### virtio-gpu is the default video model for Linux guests

Document rationale for *every* cloud_image VM author:

| Model | Pros | Cons | When to pick |
|---|---|---|---|
| `virtio` (virtio-gpu) | Native `virtio_gpu` kernel DRM driver; Wayland-native; simpler config (just `vram` + `heads`); optional `accel3d: true` for virgl OpenGL passthrough | Only modern Linux kernels (4.16+) | **default for Linux guests** |
| `qxl` | Legacy SPICE-specific (2010 Red Hat); X11-oriented; requires xf86-video-qxl for acceleration | More knobs to tune (ram_size + vram_size + vram64_size_mb + vgamem_mb quartet); simpledrm→qxldrmfb takeover race under UEFI (see `/charly-vm:arch` Finding B); X11-dominant | Legacy guests only |
| `cirrus` | Most compatible | Low resolution, no acceleration | BIOS-fallback only |
| `none` | No video | — | Headless VMs |

SPICE graphics protocol is video-model-agnostic — it streams pixels from virtio-gpu just as it does from QXL.

### PCI passthrough (`<hostdev>`) + NVIDIA Code-43 features

`mapHostdev` (`libvirt_yaml_bridge.go`) renders `libvirt.devices.hostdevs[]`. For
`type: pci` it emits `<hostdev managed='…' type='pci'><source><address …/></source>`
plus — when set — the optional `<rom bar='off'/file='…'/>` (from `rom:`) and
`<driver name='vfio'/>` (from `driver:`). `managed='yes'` makes libvirt auto-bind
the device to vfio-pci on VM start and rebind the host driver on stop. Every
function in the GPU's IOMMU group must be a separate hostdev entry (`charly vm gpu
list` emits the whole group). Passthrough wants `firmware: uefi-insecure` and the
libvirt backend (the QEMU-argv renderer skips PCI hostdev — see below).

`buildDomainFeatures` renders the two NVIDIA Code-43 workarounds for consumer
GPUs: `libvirt.features.kvm.hidden: on` → `<kvm><hidden state='on'/></kvm>`, and
`libvirt.features.hyperv.vendor_id: {state, value}` → `<hyperv><vendor_id …/>`.
`features.ibs` and the rest of the KVM struct (hint_dedicated / poll_control /
pv_ipi / dirty_ring) render too; other HyperV enlightenments stay available via
`libvirt.snippets`. `ValidateLibvirtDomain` checks hostdev type/managed enums and
hex PCI source fields. Worked example: the CachyOS `eval-cachyos-gpu-vm` bed.

### virtiofs filesystem shares + auto-paired shared memory

`mapFilesystem` (`libvirt_yaml_bridge.go`) renders `libvirt.devices.filesystems[]`
— `{driver: virtiofs|9p|path, accessmode, source: <host dir>, target: <mount tag>}`
→ `<filesystem type='mount' accessmode='…'><driver type='virtiofs'/><source dir='…'/><target dir='…'/></filesystem>`,
plus the optional virtiofsd `binary:` knobs (`path`, `xattr`, `cache`, `sandbox`,
`thread_pool` → `<binary …>`), useful for tuning the rootless `qemu:///session`
virtiofsd. **virtiofs requires shared guest memory** — `BuildLibvirtDomainXML`
calls `ensureVirtiofsSharedMemory`, which auto-injects
`<memoryBacking><source type='memfd'/><access mode='shared'/></memoryBacking>`
whenever a virtiofs share is present and no shared backing was declared (an
explicit backing, e.g. hugepages, is honored — only the missing source/access
bits are filled). Without this libvirt refuses to start the domain; auto-pairing
means an author never has to remember the coupling. `ValidateLibvirtDomain`
requires `source`+`target` and checks the driver/accessmode enums — a literal
`/home/...` source is allowed (a share's purpose is to expose a host dir). The
guest mounts the tag with the `workspace-mount` layer (or `mount -t virtiofs`).

### virtiofs guest-user idmap (auto-paired, like shared memory)

`ensureVirtiofsIdmap` (`libvirt_yaml_bridge.go`, called right after
`ensureVirtiofsSharedMemory`) auto-injects a `<filesystem><idmap>` onto every
**passthrough** virtiofs share that doesn't already declare one. libvirt's
DEFAULT rootless idmap maps **guest-root → the host operator**, so a host-home
passthrough share is `root:root` inside the guest and the interactive guest
user (uid 1000) gets `Permission denied` — the "/workspace is mounted but I
can't read it" footgun. The injected idmap instead maps the guest's primary
user (uid/gid 1000, the cloud-init/pacstrap/debootstrap first-user convention)
to the host operator, with all other ids in the operator's `/etc/subuid` +
`/etc/subgid` range:

```
<idmap>
  <uid start='0'    target='100000' count='1000'/>   <!-- guest 0-999  → subuid -->
  <uid start='1000' target='1000'   count='1'/>       <!-- guest user   → host operator -->
  <uid start='1001' target='101000' count='64536'/>   <!-- guest 1001+  → subuid -->
  <gid …/>  <!-- same partition for gids -->
</idmap>
```

Built by `guestOwnerIDMap(guestID, hostID, subStart, subCount)` from
`subIDRange("/etc/subuid", …)`. An author-declared idmap, a non-passthrough
accessmode (`mapped`/`squash`), or a missing subordinate-ID range all leave
libvirt's own default untouched. This makes "mount /home/X into the VM" usable
by the guest's interactive user, which is what the operator means in practice.

### guest-agent unix channel

`mapChannel` renders `libvirt.devices.channels[]`. The qemu-guest-agent idiom is
`{type: unix, name: org.qemu.guest_agent.0}` with **no** explicit path → a
libvirt-managed socket (`<channel type='unix'><source mode='bind'/><target
type='virtio' name='org.qemu.guest_agent.0'/></channel>`); libvirt auto-assigns
the socket path under the per-VM lib dir. `extractChannelSocketPaths` pre-creates
the parent dir. (The same channel is also contributed as a raw snippet by the
`/charly-distros:qemu-guest-agent` layer for image-composed VMs.)

## Firmware plumbing (D17)

`RenderDomain` reads `spec.Firmware` and, for UEFI, calls `ResolveOvmfForSpec` (see `/charly-internals:ovmf`) to get `(CodePath, NVRAMPath)`. Emits:

```xml
<os>
  <type arch='x86_64' machine='pc-q35-...'>hvm</type>
  <loader readonly='yes' type='pflash'>{CodePath}</loader>
  <nvram template='{template-from-ResolveOvmfForSpec}'>{NVRAMPath}</nvram>
</os>
```

When `spec.Firmware == "bios"` or empty, `ResolveOvmfForSpec` returns `("", "", nil)` and the renderer skips both `<loader>` and `<nvram>` entirely. No OVMF package dependency, no per-VM NVRAM file, no Secure Boot lock-in. This is what makes `/charly-vm:arch` viable — BIOS boot bypasses the stale BOOTX64.EFI issue by never loading it.

`uefi-secure` additionally sets `Features.SMM = true` (required for Secure Boot authenticated variables).

## SMBIOS credentials (type 11 OEM strings)

When `VmSSH.KeyInjection.SMBIOS` is enabled, the renderer emits:

```xml
<sysinfo type='smbios'>
  <oemStrings>
    <entry>io.systemd.credential:ssh.authorized_keys.root=ssh-ed25519 AAAA…</entry>
  </oemStrings>
</sysinfo>
<os>
  <smbios mode='sysinfo'/>
  ...
</os>
```

systemd-ssh-generator (systemd ≥ v250) materializes the pubkey into `~<user>/.ssh/authorized_keys` during early boot — no cloud-init needed. Runs in parallel with cloud-init's own ssh_authorized_keys injection when both channels are on; dedup happens in the guest.

## RenderQemuArgv (direct-QEMU backend)

For `charly vm create --backend qemu`, `RenderQemuArgv` emits a flat array of arguments:

```
qemu-system-x86_64 -machine pc-q35-XX,accel=kvm -cpu host,migratable=off \
  -m 8G -smp cpus=4 -drive file={disk},if=virtio,format=qcow2 \
  -cdrom {seed.iso} -device virtio-net-pci,netdev=n0 \
  -netdev user,id=n0,hostfwd=tcp::2224-:22 -display none \
  -serial mon:stdio ...
```

Intended for environments without libvirt session daemon (some CI runners, air-gapped VMs). Libvirt is the preferred backend; direct QEMU has no portForward mechanism equivalent and no SPICE console wiring.

## Validation

`ValidateLibvirtConfig` rejects:

- invalid element string in `Snippets` (checked via existing `ValidateLibvirtSnippet` — malformed XML / empty string / non-XML content).
- enumerated values out of range (e.g. `CPU.Mode` not in {`host-passthrough`, `host-model`, `custom`, `maximum`}).
- mutually exclusive combinations (e.g. `Clock.Offset` + `Clock.Adjustment` without sanity).

## Cross-References

- `/charly-internals:vm-spec` — VmSpec shape the renderer reads
- `/charly-internals:ovmf` — `ResolveOvmfForSpec` for UEFI path resolution
- `/charly-internals:cloud-init-renderer` — paired renderer for seed ISO + user-data
- `/charly-internals:vm-deploy-target` — consumer that applies the rendered domain
- `/charly-vm:vm` — command-family skill; video-model decision table
- `/charly-vm:arch` — BIOS decision RCA; virtio-gpu live-test bisect
- `/charly-distros:qemu-guest-agent` — virtio-serial channel snippet that this renderer emits in `<devices>`
