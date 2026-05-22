---
name: vm-deploy-target
description: |
  VmDeployTarget is the 4th DeployTarget implementer (after OCITarget,
  PodDeployTarget, LocalDeployTarget; K8sDeployTarget is 5th). Applies an
  InstallPlan inside a running VM over SSH. Covers DeployExecutor interface,
  SSHExecutor, ShellExecutor, VmDeployState persistence, and the guest-side
  ledger.
  Source: ov/deploy_target_vm.go, ov/deploy_executor*.go, ov/deploy_add_cmd_vm.go.
  MUST be invoked before editing VM-target deploy code.
---

# vm-deploy-target

## Implementation notes

- The pod deploy target is `PodDeployTarget` (`ov/deploy_target_pod.go`); ledger target keying uses `pod:<name>`.
- `vmNameFromDeployName` strips the `vm:` prefix. The dispatch upstream (`deploy_add_cmd.go`) rewrites a plain deploy key like `arch-vm` to `vm:<vm_source>` before calling `runVM` / `runVmDel`, so internal VM code always sees the prefixed form.
- `UnifiedDeployTarget` / `LifecycleTarget` interfaces (`ov/deploy_target_unified.go`) + the `ResolveTarget` dispatcher (`ov/unified_targets.go`) provide the full lifecycle contract (`Add` / `Del` / `Test` / `Update` / `Start` / `Stop` / `Status` / `Logs` / `Shell` / `Rebuild`).
- Disposability for `ov update <vm>` reads from the `DeploymentNode` with `target: vm` + matching `vm_source:` (see `rebuild.go::vmDisposableFromDeployments`); it is NOT a `VmSpec` field.

`VmDeployTarget` brings `ov deploy add vm:<name>` online: the same `InstallPlan` IR that drives pod builds and host deploys now runs **inside a VM** over SSH. Shell bodies that `LocalDeployTarget` would exec via local `sudo bash -s` are instead exec'd via `ssh guest 'sudo bash -s'` through an `SSHExecutor`. Ledger writes land on the **guest** filesystem under the guest user's `~/.config/overthink/installed/`; teardown runs in the guest via SSH as well.

`VmDeployTarget` is the 4th `DeployTarget` interface implementer — after `OCITarget` (build-mode Containerfile emission), `PodDeployTarget` (podman quadlet), and `LocalDeployTarget` (local filesystem). `K8sDeployTarget` is the 5th. See `/ov-internals:install-plan` for the shared IR.

## Source files

| File | Contents |
|---|---|
| `ov/deploy_target_vm.go` | `VmDeployTarget` struct + `Emit` flow |
| `ov/deploy_executor.go` | `DeployExecutor` interface (RunShell, Scp, Close) + `ShellExecutor` — local shell exec (reused by `LocalDeployTarget` for the builder-image step) |
| `ov/deploy_executor_ssh.go` | `SSHExecutor` — ssh client with passt-friendly timeouts + WaitForSSH + WaitForCloudInit |
| `ov/deploy_add_cmd_vm.go` | CLI dispatch: `runVM` + `runVmDel` for `ov deploy add/del vm:<name>` |
| `ov/vm_create_spec.go` | `VmCreateCmd.runVmSpecCreate` — prereq: VM must be created before deploy |

## DeployExecutor interface

```go
type DeployExecutor interface {
    RunShell(ctx context.Context, script string, opts ShellOpts) (ExecResult, error)
    Scp(ctx context.Context, src io.Reader, dst string, mode os.FileMode) error
    Close() error
}
```

Two implementations:

- `ShellExecutor` — `bash -c <script>` / file copy. Used by `LocalDeployTarget` for container-builder invocations and by the dry-run path of any target.
- `SSHExecutor` — ssh/scp via `golang.org/x/crypto/ssh`. Used exclusively by `VmDeployTarget`. Carries Host/Port/User/KeyPath + maintains a persistent connection across multiple shell invocations.

**Name choice**: the interface is `DeployExecutor` (not `Executor`) to avoid a clash with the `Executor` type in `testrun.go`; likewise `deployShellQuote` (not `shellQuote`) avoids a clash in `wl.go`.

## VmDeployTarget.Emit flow

Five preflight steps before walking plans:

1. **Wait for SSH.** `SSHExecutor.WaitForSSH(ctx, 120)` — polls `net.Dial` to `host:port` with exponential backoff. 120s timeout accommodates cold-boot VMs where cloud-init is provisioning sshd.
2. **Wait for cloud-init** (cloud_image sources only). `SSHExecutor.WaitForCloudInit` polls `cloud-init status --wait` until status is `done`. Bootc guests skip this step unless the `cloud-init` layer is present.
3. **EnsureOvInGuest.** Runs the `VmOvInstall.Strategy` state machine (see `/ov-internals:cloud-init-renderer`).
4. **Ensure guest ledger dir exists.** `ssh -- mkdir -p ~/.config/overthink/installed/{deploys,layers}`.
5. **Walk plans.** Same batched `(Scope, Venue)` logic as `LocalDeployTarget`, but with `sudo bash -s` wrapped in `ssh`. See `/ov-local:local-deploy` for the grouping rules.

## VmDeployState persistence

```go
type VmDeployState struct {
    InstanceID              string                  // stable UUIDv4 cloud-init instance-id, pinned across re-renders
    DiskPath                string                  // absolute path to the qcow2 (may be a CoW overlay on a cached base)
    SeedIso                 string                  // NoCloud cidata ISO path (empty for bootc with injection disabled)
    SshPort                 int                     // host port forwarded to the guest's :22
    SshUser                 string                  // guest account VmDeployTarget SSHes in as
    Backend                 string                  // "qemu" or "libvirt", pinned at first apply
    KeyInjectionResolved    *VmKeyInjectionResolved // resolved SSH key-injection plan
    OvInstallStrategy       string                  // how ov is installed into the guest
    CloudInitRenderedDigest string                  // digest of the rendered cloud-init (re-render detection)
    Snapshots               []VmSnapshotState       // libvirt snapshot ledger
    Ephemeral               *EphemeralRuntime       // transient run-state for an ephemeral VM
}
```

Persisted in `~/.config/ov/deploy.yml` as the `vm_state:` field on the VM's deploy entry (`DeploymentNode.VmState`). Each `ov vm build` / `ov vm create` / `ov deploy add vm:<name>` iteration updates the relevant fields. `ov deploy del vm:<name>` preserves the state (so re-adding picks up InstanceID etc.) unless `--purge` is passed.

## SSH key idempotency

`generateSSHKeypair` in `ov/vm.go` checks for `<vmStateDir>/id_ed25519.pub` before creating. Rebuilding a VM doesn't regenerate the keypair. First `ov vm build` writes the keypair; subsequent calls leave it untouched — so iterated rebuilds keep a stable pubkey and SSH stays valid.

## CLI dispatch: runVM / runVmDel

`deploy_add_cmd_vm.go::runVM` is called when the deploy name starts with `vm:` (or `target: vm` is set in `deploy.yml`):

```
ov deploy add vm:arch ripgrep           # apply ripgrep layer in the guest
ov deploy add vm:arch fedora-coder \    # apply full fedora-coder layer set
    --add-layer team-extras \
    --add-layer github.com/team/configs/layers/sshkeys
ov deploy del vm:arch                   # reverse all applied layers in the guest
```

Prereq: VM must exist (`ov vm create arch` first). `runVM` does NOT auto-provision the VM — keeps the "provision" step explicit. If the VM is undefined, the dispatch returns a clean error pointing at `ov vm create`.

## passt backend + SSH port forwarding

When the VM's network uses libvirt user-mode + `<backend type='passt'/>` + `<portForward>` (see `/ov-internals:libvirt-renderer`), SSHExecutor connects to `127.0.0.1:<host-port>`. The portForward maps that through passt into the guest's `:22`. The indirection is invisible to SSHExecutor — it sees a normal TCP connect.

## Cross-References

- `/ov-internals:install-plan` — InstallPlan IR (the 4 DeployTarget implementers and the 9 step kinds)
- `/ov-internals:vm-spec` — VmSpec consumed by VmDeployTarget
- `/ov-internals:libvirt-renderer` — renders domain XML; portForward + passt backend
- `/ov-internals:cloud-init-renderer` — `EnsureOvInGuest` lives there
- `/ov-core:deploy` — `ov deploy add vm:<name>` command + deploy.yml schema
- `/ov-local:local-deploy` — parallel target (LocalDeployTarget); ReverseOps model also used on VM target
- `/ov-vm:vm` — VM lifecycle; creates the target Emit runs against
- `/ov-vm:arch` — canonical worked example — VmDeployState persistence; ssh_key idempotency live-test
