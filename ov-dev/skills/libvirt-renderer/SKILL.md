---
name: libvirt-renderer
description: |
  Pure renderer from VmSpec + LibvirtConfig to libvirt domain XML and QEMU argv.
  Covers RenderDomain, device emission (passt backend, portForward attribute
  order, virtio-gpu defaults), firmware plumbing, and LibvirtConfig schema shape.
  Source: ov/libvirt_schema.go, ov/libvirt_render.go, ov/libvirt_render_devices.go,
  ov/qemu_render.go. MUST be invoked before editing libvirt XML emission.
---

# libvirt-renderer

The libvirt renderer converts `VmSpec` + `LibvirtConfig` into a libvirt domain XML (`RenderDomain`) and — for the direct-QEMU backend — into a QEMU argv array (`RenderQemuArgv`). Pure functions: given the same inputs, they produce the same output; no side effects. Consumed by `ov vm create` (libvirt path) and `ov vm create --backend qemu` (direct path).

## Source files

| File | Contents | LOC |
|---|---|---|
| `ov/libvirt_schema.go` | `LibvirtConfig` + 30+ sub-types (features, CPU, clock, memory backing, memtune, numatune, cputune, devices, seclabel, launch security, resource, sysinfo) | ~470 |
| `ov/libvirt_render.go` | `RenderDomain` top-level composition; firmware plumbing (D17); SMBIOS credentials | ~800 |
| `ov/libvirt_render_devices.go` | `<devices>` child emitters — channels, graphics, video, rng, memballoon, hostdev, interface (with portForward), filesystem | ~700 |
| `ov/qemu_render.go` | `RenderQemuArgv` for direct-QEMU backend | ~340 |
| `ov/libvirt_validate.go` | `ValidateLibvirtConfig` |

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

**Structured first; raw XML is a last resort.** `Snippets` exists for the rare case where libvirt gained a new element before ov's schema caught up. The legacy list-of-strings form `libvirt: ["<xml>", ...]` on `kind:image` entries was deleted in the cutover — raw XML now lives only in `vms.<name>.libvirt.snippets:` (new), on layer-level `libvirt.snippets:` fields, and the `InjectLibvirtXML` post-processor still handles both paths.

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
| `qxl` | Legacy SPICE-specific (2010 Red Hat); X11-oriented; requires xf86-video-qxl for acceleration | More knobs to tune (ram_size + vram_size + vram64_size_mb + vgamem_mb quartet); simpledrm→qxldrmfb takeover race under UEFI (see `/ov-vms:arch-cloud-base` Finding B); X11-dominant | Legacy guests only |
| `cirrus` | Most compatible | Low resolution, no acceleration | BIOS-fallback only |
| `none` | No video | — | Headless VMs |

SPICE graphics protocol is video-model-agnostic — it streams pixels from virtio-gpu just as it does from QXL.

## Firmware plumbing (D17)

`RenderDomain` reads `spec.Firmware` and, for UEFI, calls `ResolveOvmfForSpec` (see `/ov-dev:ovmf`) to get `(CodePath, NVRAMPath)`. Emits:

```xml
<os>
  <type arch='x86_64' machine='pc-q35-...'>hvm</type>
  <loader readonly='yes' type='pflash'>{CodePath}</loader>
  <nvram template='{template-from-ResolveOvmfForSpec}'>{NVRAMPath}</nvram>
</os>
```

When `spec.Firmware == "bios"` or empty, `ResolveOvmfForSpec` returns `("", "", nil)` and the renderer skips both `<loader>` and `<nvram>` entirely. No OVMF package dependency, no per-VM NVRAM file, no Secure Boot lock-in. This is what makes `/ov-vms:arch-cloud-base` viable — BIOS boot bypasses the stale BOOTX64.EFI issue by never loading it.

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

For `ov vm create --backend qemu`, `RenderQemuArgv` emits a flat array of arguments:

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

- `/ov-dev:vm-spec` — VmSpec shape the renderer reads
- `/ov-dev:ovmf` — `ResolveOvmfForSpec` for UEFI path resolution
- `/ov-dev:cloud-init-renderer` — paired renderer for seed ISO + user-data
- `/ov-dev:vm-deploy-target` — consumer that applies the rendered domain
- `/ov:vm` — command-family skill; video-model decision table
- `/ov-vms:arch-cloud-base` — BIOS decision RCA; virtio-gpu live-test bisect
- `/ov-layers:qemu-guest-agent` — virtio-serial channel snippet that this renderer emits in `<devices>`
