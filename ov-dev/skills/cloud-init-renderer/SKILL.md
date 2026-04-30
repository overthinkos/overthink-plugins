---
name: cloud-init-renderer
description: |
  Pure renderer from VmSpec + VmCloudInit to NoCloud seed ISO (user-data +
  meta-data + network-config). Covers composeUsers adopt-merge, SMBIOS vs
  cloud_init additive channels, xorriso ISO emission, and ov_install.strategy
  state machine. Source: ov/cloud_init_render.go, ov/cloud_init_iso.go, ov/ov_install.go.
  MUST be invoked before editing cloud-init emission paths.
---

# cloud-init-renderer

Host-side renderer producing NoCloud seed ISOs for cloud_image VMs (and bootc VMs that include the `cloud-init` layer). Pure transformation — given `VmSpec` + `VmCloudInit`, produces three files (user-data, meta-data, network-config) and packages them into a FAT-labeled `cidata` ISO via xorriso.

Lives **host-side**, in the `ov` binary. The **guest-side** `/ov-foundation:cloud-init` layer is complementary: it puts the cloud-init package into the bootc guest OS so that guest reads the seed ISO. The two sides cooperate across the host/guest boundary.

## Source files

| File | Contents |
|---|---|
| `ov/cloud_init_render.go` | `RenderCloudInit`, `ResolveKeyInjectionChannels`, `composeUsers`, `composePackages`, `composeRunCmd` |
| `ov/cloud_init_iso.go` | `WriteSeedISO` via xorriso; `genisoimage` + `mkisofs` fallbacks |
| `ov/ov_install.go` | `EnsureOvInGuest` state machine (auto/scp/url/skip strategies) |
| `ov/cloud_init_types.go` | `VmCloudInit`, `VmCloudInitUser`, `VmCloudInitFile`, `VmCloudInitNetwork`, `VmCloudInitMirrors`, `VmOvInstall` |

## RenderCloudInit top-level

```go
func RenderCloudInit(spec *VmSpec, vmName, sshPubKey string) (userData, metaData, networkConfig string, err error)
```

Returns three strings — the three files NoCloud expects on the seed ISO. The caller (`BuildCloudImage` or `VmDeployTarget`) then calls `WriteSeedISO` to pack them.

## composeUsers (adopt-merge pattern)

Single most important function in this renderer. Produces the `users:` list in user-data.

**When `spec.Source.BaseUser` is non-empty** (cloud_image adopt pattern):

```yaml
users:
  - default                  # cloud-init sentinel: preserve distro's default account
  - name: <base_user>
    ssh_authorized_keys:
      - <pubkey>
```

No `useradd`, no sudoers write, no shell change — cloud-init interprets "name: X" on an existing account as "append ssh_authorized_keys". The parity is exact with the container-side `base_user:` + `user_policy: adopt` pattern (`/ov-build:image` "user_policy").

**When `BaseUser` is empty and `VmSSH.User` is non-empty** (create pattern):

```yaml
users:
  - default
  - name: <ssh_user>
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: [wheel, sudo]
    shell: /bin/bash
    lock_passwd: true
    ssh_authorized_keys:
      - <pubkey>
```

Full account provisioning. Used by bootc VMs where no `base_user:` applies.

**When a user already appears in `spec.CloudInit.Users`**: the renderer appends the ssh pubkey to that existing entry instead of emitting a new one. Lets authors declare a user with specific sudo/groups/shell fields and still get the pubkey injected.

## ResolveKeyInjectionChannels

Applies the per-source-kind auto-defaults documented in `/ov-dev:vm-spec`:

```go
func ResolveKeyInjectionChannels(spec *VmSpec) (smbios bool, cloudInit bool) {
    if spec.SSH != nil && spec.SSH.KeyInjection != nil {
        // explicit overrides
        return spec.SSH.KeyInjection.SMBIOS == "enabled" (with "auto" = source-kind default),
               spec.SSH.KeyInjection.CloudInit == "enabled" (with "auto" = source-kind default)
    }
    // per-source-kind auto-defaults
    switch spec.Source.Kind {
    case "cloud_image":
        return true, true       // belt + suspenders
    case "bootc":
        return true, false      // cloud_init channel only activates with cloud-init layer
    }
}
```

**Both channels are additive** — when both are on, systemd-ssh-generator (SMBIOS path) and cloud-init (user-data path) both inject the key. Dedup happens in the guest's `authorized_keys`. There's no correctness issue with duplicate keys; the dual-injection pattern is the safe default.

## composePackages + composeRunCmd renderer defaults

The renderer prepends defaults to user-declared lists:

- `composePackages`: prepends `{openssh, curl, tar}` (deduplicated against user's `Packages`). Guarantees the guest has SSH server + download tools + tar for later layer application.
- `composeRunCmd`: prepends `{systemctl enable --now sshd}`. Distro-specific user `runcmd:` entries can assume sshd is running.

User-supplied fields **extend** defaults; they don't replace them. Prevents the common footgun where an author puts `packages: [nginx]` and accidentally breaks SSH because they overrode the default list.

## WriteSeedISO

```go
func WriteSeedISO(userData, metaData, networkConfig string, outputPath string) error
```

Writes a FAT-labeled `cidata` ISO. Tool preference order:

1. `xorriso` (preferred — modern, scriptable).
2. `genisoimage` (legacy but widely available).
3. `mkisofs` (oldest fallback).

Clean error when none are present, with distro-appropriate install recipe (`dnf install xorriso`, `pacman -S libisoburn`, `apt-get install xorriso`).

The ISO is mounted by QEMU as a CD-ROM; cloud-init's NoCloud datasource reads `/dev/sr0` at first boot.

## EnsureOvInGuest (ov_install.strategy state machine)

Runs post-boot inside `VmDeployTarget.Emit` after cloud-init completes. Dispatches on `spec.CloudInit.OvInstall.Strategy`:

| Strategy | Action |
|---|---|
| `auto` / `scp` | `scp $(os.Executable()) guest:/usr/local/bin/ov; chmod +x` |
| `url` | ssh-execute `curl -L <url> -o /usr/local/bin/ov && sha256sum -c` (verified against `VmOvInstall.Checksum`) |
| `skip` | `ssh 'which ov'` — fails if missing, returns early if present |

Idempotent. If `ov` is already present at the target version, the function returns without re-scp'ing. See `/ov-dev:vm-deploy-target` for how this plugs into the overall deploy flow.

## Network config

`VmCloudInitNetwork.Ethernets` passes through to cloud-init's network-config v2 as-is. When unset, the renderer emits an empty network-config (cloud-init defaults to DHCP on every virtio-net interface). Good default; override only for static-IP deployments.

## SMBIOS vs cloud_init can coexist

Explicitly supported — not either/or. `VmKeyInjection.SMBIOS: enabled` + `VmKeyInjection.CloudInit: enabled` simultaneously is the default for cloud_image VMs. Rationale: belt-and-suspenders (SMBIOS via systemd-ssh-generator v250+, cloud-init via user-data — both paths exist on modern Linux). No duplication cost.

## Cross-References

- `/ov-dev:vm-spec` — `VmCloudInit`, `VmSSH.KeyInjection`, `VmOvInstall` types
- `/ov-dev:libvirt-renderer` — SMBIOS-channel emission (domain XML side)
- `/ov-dev:vm-deploy-target` — `EnsureOvInGuest` caller; SSH/cloud-init readiness waits
- `/ov-advanced:vm` — command-family; cloud-init flow
- `/ov-vms:vms` — YAML-authoring reference
- `/ov-vms:arch` — `ov_install.strategy: auto` worked example; adopt-user pattern
- `/ov-foundation:cloud-init` — **guest-side pairing**: the cloud-init package installed inside bootc images reads the seed ISO this renderer produces
