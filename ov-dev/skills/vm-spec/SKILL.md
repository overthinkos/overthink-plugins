---
name: vm-spec
description: |
  Go type reference for VmSpec and the discriminated-union source types
  (VmSource cloud_image | bootc). Documents every field, validation rules,
  and the adopt-user decision. Source files: ov/vm_spec.go, ov/cloud_init_types.go,
  ov/libvirt_validate.go.
  MUST be invoked before editing VmSpec Go code or authoring vms.yml entries.
---

# vm-spec

Go type reference for the VM surface. `VmSpec` + `VmSource` + `VmChecksum` + `VmNetwork` + `VmSSH` + `VmKeyInjection` + `VmCloudInit` + `VmOvInstall` are the canonical types that drive `ov vm build`, `ov vm create`, and `ov deploy add vm:<name>`. This skill is the authoritative Go reference — field semantics, defaults, validation rules, migration history. The YAML-authoring companion is `/ov-vms:vms`.

## Source files

| File | Contents |
|---|---|
| `ov/vm_spec.go` | `VmSpec`, `VmSource`, `VmChecksum`, `VmNetwork`, `VmSSH`, `VmKeyInjection` |
| `ov/cloud_init_types.go` | `VmCloudInit`, `VmCloudInitUser`, `VmCloudInitFile`, `VmCloudInitNetwork`, `VmCloudInitMirrors`, `VmOvInstall` |
| `ov/libvirt_schema.go` | `LibvirtConfig` + 30+ sub-types (features, CPU, clock, devices, etc.) |
| `ov/libvirt_validate.go` | `ValidateVmSpec`, `ValidateLibvirtConfig` |

## VmSpec (top-level shape)

```go
type VmSpec struct {
    Source VmSource       // discriminated union: kind = cloud_image | bootc
    DiskSize string       // "20G", "10 GiB"
    Ram      string       // "4G", "8192M"
    Cpus     int
    Machine  string       // q35 | virt | i440fx — default: host-native
    Firmware string       // bios | uefi-insecure | uefi-secure — default: bios
    Network  *VmNetwork
    SSH      *VmSSH
    CloudInit *VmCloudInit
    Libvirt  *LibvirtConfig
}
```

Every field except `Source`, `CloudInit`, and `Source`-branch-specific subfields applies equally to both source kinds — the parity guarantee. Anything configurable for cloud_image VMs is configurable for bootc VMs, and vice versa.

## VmSource (discriminated union)

```go
type VmSource struct {
    Kind string    // "cloud_image" | "bootc"

    // cloud_image branch:
    URL      string
    Checksum VmChecksum
    Cache    string
    BaseUser string       // adopt-user pattern — see below

    // bootc branch:
    Image      string     // kind:image entry name
    Transport  string     // registry | containers-storage | oci | oci-archive
    Rootfs     string     // ext4 | xfs | btrfs
    RootSize   string     // "10G" — caps root partition, rest unpartitioned
    KernelArgs string
}
```

`Kind` is the discriminator. `ValidateVmSpec` enforces that exactly one branch's required fields are populated.

### Adopt-user decision (BaseUser)

`BaseUser` mirrors the container-side `base_user:` + `user_policy: adopt` pattern. When set:

1. `/ov-dev:cloud-init-renderer::composeUsers` emits `users: [default, {name: <base_user>, ssh_authorized_keys: [...]}]` — merge-by-name, no `useradd`.
2. `spec.ssh.user` defaults to `BaseUser`.
3. cloud-init appends the pubkey to `~<base_user>/.ssh/authorized_keys` without touching sudoers/shell/home.

Common values: `arch` (Arch cloud image), `ubuntu` (Ubuntu Cloud), `fedora` (Fedora Cloud), `debian` (Debian Cloud), `cloud-user` (CentOS Cloud).

Leave empty **only** when the image has no default account — then declare a custom user in `spec.cloud_init.users`.

## VmSSH / VmKeyInjection

```go
type VmSSH struct {
    User         string   // default: cloud_image → "ov" OR base_user; bootc → "root"
    Port         int      // default: 2222
    KeySource    string   // auto | generate | none | <abs-path>
    KeyInjection *VmKeyInjection
}

type VmKeyInjection struct {
    SMBIOS    string  // auto | enabled | disabled
    CloudInit string  // auto | enabled | disabled
}
```

**Dual-channel key injection is additive.** Per-source-kind auto-defaults when `KeyInjection` is nil:

- `cloud_image` → `{smbios: enabled, cloud_init: enabled}` — belt + suspenders; cloud-init seed ISO is always emitted anyway.
- `bootc` → `{smbios: enabled, cloud_init: disabled}` — cloud-init seed ISO only emits when the guest has the `cloud-init` layer.

Having both channels on simultaneously is the safe default; there's no duplication cost at the guest (systemd-ssh-generator dedups entries in `authorized_keys`).

## VmCloudInit (structured intent)

The renderer combines structured fields with renderer defaults:

- `Packages`: prepended with `{openssh, curl, tar}`.
- `RunCmd`: prepended with `{systemctl enable --now sshd}` so distro-specific setup can assume sshd is running.
- `Users`: the `VmSSH.User` account is auto-injected unless already present; if present, the renderer appends the ssh pubkey to the user's existing entry.

`Extra` is a raw-YAML escape hatch merged after structured fields. Prefer structured fields; `Extra` exists for long-tail cloud-init options the schema doesn't cover.

## VmOvInstall strategies

```go
type VmOvInstall struct {
    Strategy string  // auto | scp | url | skip
    URL      string  // when Strategy == "url"
    Checksum string  // "sha256:<hex>" for url strategy
}
```

| Strategy | Behavior |
|---|---|
| `auto` (default) | scp the local `ov` binary (`os.Executable()`) into the guest post-boot via VmDeployTarget |
| `scp` | explicit form of auto |
| `url` | cloud-init runcmd downloads ov from URL at first boot |
| `skip` | user manages ov install; VmDeployTarget verifies presence only |

## Validation (ov/libvirt_validate.go)

`ValidateVmSpec` enforces the invariants documented in `/ov-vms:vms`. Key checks:

- `source.kind` ∈ {`cloud_image`, `bootc`}.
- cloud_image branch requires `url:` populated; bootc branch requires `image:`.
- `firmware:` ∈ {`bios`, `uefi-insecure`, `uefi-secure`}.
- `network.mode:` ∈ {`user`, `bridge`, `nat`}.
- `ssh.key_source:` parses as `auto` | `generate` | `none` | absolute path.
- `ssh.key_injection.{smbios,cloud_init}` ∈ {`auto`, `enabled`, `disabled`}.
- `Libvirt` structure routed to `ValidateLibvirtConfig`.

Failed validation → hard load-time error with a one-line remediation hint pointing at `/ov-vms:vms` or `ov migrate vm-spec`.

## Migration from legacy VmConfig

The legacy `VmConfig` type + `ImageConfig.Vm` + `ImageConfig.Libvirt` + `ResolvedImage.Vm` + `LabelVm` + `LabelLibvirt` were **all deleted** in the hard cutover. Field mapping for forensic purposes:

| Legacy location | New location |
|---|---|
| `image.bootc: true` + `image.vm.disk_size` | `vms.<name>.source.kind: bootc` + `vms.<name>.disk_size` |
| `image.vm.ssh_port` | `vms.<name>.ssh.port` |
| `image.vm.ram`, `.cpus`, `.rootfs`, `.root_size`, `.kernel_args` | `vms.<name>.ram`, `.cpus`, `source.rootfs`, `source.root_size`, `source.kernel_args` |
| `image.vm.firmware` | `vms.<name>.firmware` |
| `image.vm.network` (string) | `vms.<name>.network.mode` |
| `image.libvirt: ["<xml>", …]` (list of strings) | `vms.<name>.libvirt.snippets: […]` + structured `libvirt.devices.*` |

`ov migrate vm-spec` performs this mapping idempotently. See `/ov:migrate` for the command and `/ov-dev:cutover-policy` for the policy.

## Cross-References

- `/ov-vms:vms` — YAML-authoring companion (when to pick cloud_image vs bootc, adopt pattern, step-by-step recipes)
- `/ov-dev:libvirt-renderer` — `LibvirtConfig` rendering + pure render functions
- `/ov-dev:cloud-init-renderer` — `RenderCloudInit`, `composeUsers`, seed ISO
- `/ov-dev:vm-deploy-target` — VmDeployTarget consuming VmSpec via DeployExecutor
- `/ov-dev:ovmf` — `ResolveOvmfForSpec` reads `spec.Firmware`
- `/ov-dev:cutover-policy` — why the legacy surface was deleted in one PR
- `/ov:vm` — command-family skill; reads vms.yml through VmSpec
- `/ov:migrate` — `ov migrate vm-spec` command
- `/ov-dev:go` — Go CLI development overview
