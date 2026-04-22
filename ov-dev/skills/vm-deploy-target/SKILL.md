---
name: vm-deploy-target
description: |
  VmDeployTarget is the 4th DeployTarget implementer (after OCITarget,
  ContainerDeployTarget, HostDeployTarget; K8s is 5th). Applies an InstallPlan
  inside a running VM over SSH. Covers DeployExecutor interface, SSHExecutor,
  LocalExecutor, VmDeployState persistence, and the guest-side ledger.
  Source: ov/deploy_target_vm.go, ov/deploy_executor*.go, ov/deploy_add_cmd_vm.go.
  MUST be invoked before editing VM-target deploy code.
---

# vm-deploy-target

`VmDeployTarget` brings `ov deploy add vm:<name>` online: the same `InstallPlan` IR that drives container builds and host deploys now runs **inside a VM** over SSH. Shell bodies that `HostDeployTarget` would exec via local `sudo bash -s` are instead exec'd via `ssh guest 'sudo bash -s'` through an `SSHExecutor`. Ledger writes land on the **guest** filesystem under the guest user's `~/.config/overthink/installed/`; teardown runs in the guest via SSH as well.

`VmDeployTarget` is the 4th `DeployTarget` interface implementer — after `OCITarget` (build-mode Containerfile emission), `ContainerDeployTarget` (podman quadlet), and `HostDeployTarget` (local filesystem). `KubernetesDeployTarget` is the 5th. See `/ov-dev:install-plan` for the shared IR.

## Source files

| File | Contents |
|---|---|
| `ov/deploy_target_vm.go` | `VmDeployTarget` struct + `Emit` flow |
| `ov/deploy_executor.go` | `DeployExecutor` interface (RunShell, Scp, Close) |
| `ov/deploy_executor_local.go` | `LocalExecutor` — local shell exec (reused by HostDeployTarget for the builder-image step) |
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

- `LocalExecutor` — `bash -c <script>` / file copy. Used by `HostDeployTarget` for container-builder invocations and by the dry-run path of any target.
- `SSHExecutor` — ssh/scp via `golang.org/x/crypto/ssh`. Used exclusively by `VmDeployTarget`. Carries Host/Port/User/KeyPath + maintains a persistent connection across multiple shell invocations.

**Name-clash history**: originally named `Executor`; renamed to `DeployExecutor` because `testrun.go:68` already owned `Executor`. Same pattern for `shellQuote` → `deployShellQuote` (clashed with `wl.go:1537`). Small Go-level detail that saved a downstream merge conflict.

## VmDeployTarget.Emit flow

Five preflight steps before walking plans:

1. **Wait for SSH.** `SSHExecutor.WaitForSSH(ctx, 120)` — polls `net.Dial` to `host:port` with exponential backoff. 120s timeout accommodates cold-boot VMs where cloud-init is provisioning sshd.
2. **Wait for cloud-init** (cloud_image sources only). `SSHExecutor.WaitForCloudInit` polls `cloud-init status --wait` until status is `done`. Bootc guests skip this step unless the `cloud-init` layer is present.
3. **EnsureOvInGuest.** Runs the `VmOvInstall.Strategy` state machine (see `/ov-dev:cloud-init-renderer`).
4. **Ensure guest ledger dir exists.** `ssh -- mkdir -p ~/.config/overthink/installed/{deploys,layers}`.
5. **Walk plans.** Same batched `(Scope, Venue)` logic as `HostDeployTarget`, but with `sudo bash -s` wrapped in `ssh`. See `/ov:host-deploy` "HostDeployTarget execution model" for the grouping rules.

## VmDeployState persistence

```go
type VmDeployState struct {
    InstanceID   string           // stable UUID — stays the same across rebuilds
    SshKeyPath   string           // absolute path on host, e.g. ~/.local/share/ov/vm/ov-arch-cloud-base/id_ed25519
    NvramPath    string           // absolute path, empty for firmware=bios
    LastBuild    time.Time
    LastDeploy   time.Time
    AppliedLayers []string         // layer names applied inside the guest
    BaseImageSHA256 string         // for cloud_image — integrity trace
}
```

Persisted in `~/.config/ov/deploy.yml` under `images[<deploy-name>].vm_state`. Each `ov vm build` / `ov vm create` / `ov deploy add vm:<name>` iteration updates the relevant fields. `ov deploy del vm:<name>` preserves the state (so re-adding picks up InstanceID etc.) unless `--purge` is passed.

## SSH key idempotency

`generateSSHKeypair` in `ov/vm_cloud_image.go` checks for `<vmStateDir>/id_ed25519.pub` before creating. Rebuilding a VM doesn't regenerate the keypair. First `ov vm build` writes the keypair; subsequent calls leave it untouched. Caught in the live-test phase when iterated rebuilds kept rotating the pubkey and breaking SSH.

## CLI dispatch: runVM / runVmDel

`deploy_add_cmd_vm.go::runVM` is called when the deploy name starts with `vm:` (or `target: vm` is set in `deploy.yml`):

```
ov deploy add vm:arch-cloud-base ripgrep           # apply ripgrep layer in the guest
ov deploy add vm:arch-cloud-base fedora-coder \    # apply full fedora-coder layer set
    --add-layer team-extras \
    --add-layer github.com/team/configs/layers/sshkeys
ov deploy del vm:arch-cloud-base                   # reverse all applied layers in the guest
```

Prereq: VM must exist (`ov vm create arch-cloud-base` first). `runVM` does NOT auto-provision the VM — keeps the "provision" step explicit. If the VM is undefined, the dispatch returns a clean error pointing at `ov vm create`.

## passt backend + SSH port forwarding

When the VM's network uses libvirt user-mode + `<backend type='passt'/>` + `<portForward>` (see `/ov-dev:libvirt-renderer`), SSHExecutor connects to `127.0.0.1:<host-port>`. The portForward maps that through passt into the guest's `:22`. The indirection is invisible to SSHExecutor — it sees a normal TCP connect.

## Cross-References

- `/ov-dev:install-plan` — InstallPlan IR (the 4 DeployTarget implementers and the 8 step kinds)
- `/ov-dev:vm-spec` — VmSpec consumed by VmDeployTarget
- `/ov-dev:libvirt-renderer` — renders domain XML; portForward + passt backend
- `/ov-dev:cloud-init-renderer` — `EnsureOvInGuest` lives there
- `/ov:deploy` — `ov deploy add vm:<name>` command + deploy.yml schema
- `/ov:host-deploy` — parallel target (HostDeployTarget); ReverseOps model also used on VM target
- `/ov:vm` — VM lifecycle; creates the target Emit runs against
- `/ov-vms:arch-cloud-base` — canonical worked example — VmDeployState persistence; ssh_key idempotency live-test
