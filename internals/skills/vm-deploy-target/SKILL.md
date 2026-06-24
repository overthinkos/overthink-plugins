---
name: vm-deploy-target
description: |
  VmDeployTarget is the 4th DeployTarget implementer (after OCITarget,
  PodDeployTarget, LocalDeployTarget; K8sDeployTarget is 5th). Applies an
  InstallPlan inside a running VM over SSH. Covers DeployExecutor interface,
  SSHExecutor, ShellExecutor, VmDeployState persistence, and the guest-side
  ledger.
  Source: charly/deploy_target_vm.go, charly/deploy_executor*.go, charly/deploy_add_cmd_vm.go.
  MUST be invoked before editing VM-target deploy code.
---

# vm-deploy-target

## Implementation notes

- The pod deploy target is `PodDeployTarget` (`charly/deploy_target_pod.go`); ledger target keying uses `pod:<name>`.
- `vmNameFromDeployName` strips the `vm:` prefix. The dispatch upstream (`deploy_add_cmd.go`) rewrites a plain deploy key like `check-arch-vm` to `vm:<vm-name>` before resolving via `ResolveTarget` → `VmUnifiedTarget.Add` / `.Del`, so internal VM code always sees the prefixed form.
- `UnifiedDeployTarget` / `LifecycleTarget` interfaces (`charly/deploy_target_unified.go`) + the `ResolveTarget` dispatcher (`charly/unified_targets.go`) provide the full lifecycle contract (`Add` / `Del` / `Test` / `Update` / `Start` / `Stop` / `Status` / `Logs` / `Shell` / `Rebuild`).
- Disposability is read per-`BundleNode` via `charly/deploy.go::BundleNode.IsDisposable()` (`disposable: true`, or ephemeral); it is NOT a `VmSpec` field. The disposability-as-authorization gate is NOT applied in the `charly update` path — `charly update <vm>` rebuilds on explicit invocation regardless (it only NOTES non-disposability, never refuses). `VmUnifiedTarget.Rebuild` (`charly/unified_targets_vm.go`) recreates the domain THEN re-applies the deploy node's layers via the shared `charly bundle add <node>` path — the same layer-apply primitive `LocalUnifiedTarget`/`PodUnifiedTarget` Rebuild use (R3).

`VmDeployTarget` brings `charly bundle add vm:<name>` online: the same `InstallPlan` IR that drives pod builds and host deploys now runs **inside a VM** over SSH. Shell bodies that `LocalDeployTarget` would exec via local `sudo bash -s` are instead exec'd via `ssh guest 'sudo bash -s'` through an `SSHExecutor`. Ledger writes land on the **guest** filesystem under the guest user's `~/.config/opencharly/installed/`; teardown runs in the guest via SSH as well.

`VmDeployTarget` is the 4th `DeployTarget` interface implementer — after `OCITarget` (pod-overlay `add_candy:` Containerfile synthesis; `charly box build`/`generate` itself uses the separate `writeCandySteps` → `emitTasks` generator), `PodDeployTarget` (podman quadlet), and `LocalDeployTarget` (local filesystem). `K8sDeployTarget` is the 5th. See `/charly-internals:install-plan` for the shared IR.

## Source files

| File | Contents |
|---|---|
| `charly/deploy_target_vm.go` | `VmDeployTarget` struct + `Emit` flow |
| `charly/deploy_executor.go` | `DeployExecutor` interface (RunShell, Scp, Close) + `ShellExecutor` — local shell exec (reused by `LocalDeployTarget` for the builder-image step) |
| `charly/deploy_executor_ssh.go` | `SSHExecutor` — ssh client with passt-friendly timeouts + WaitForSSH + WaitForCloudInit |
| `charly/deploy_add_cmd_vm.go` | VM-only deploy helpers (`deployNestedPodsInGuest`, `buildVmReverseRunner`, `vmNameFromDeployName`); `charly bundle add/del vm:<name>` dispatches through `ResolveTarget` → `VmUnifiedTarget.Add` / `.Del` |
| `charly/vm_create_spec.go` | `VmCreateCmd.runVmSpecCreate` — prereq: VM must be created before deploy |

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

**Name choice**: the interface is `DeployExecutor` — a deploy-scoped name kept distinct from the check runner's own execution types; likewise `deployShellQuote` (not `shellQuote`) avoids a clash in `wl.go`.

## VmDeployTarget.Emit flow

Five preflight steps before walking plans:

1. **Wait for SSH.** `SSHExecutor.WaitForSSH(ctx, 120)` — polls `net.Dial` to `host:port` with exponential backoff. 120s timeout accommodates cold-boot VMs where cloud-init is provisioning sshd.
2. **Wait for cloud-init** (cloud_image sources only). `SSHExecutor.WaitForCloudInit` polls `cloud-init status --wait` until status is `done`. Bootc guests skip this step unless the `cloud-init` layer is present.
3. **EnsureCharlyInGuest.** Runs the `VmCharlyInstall.Strategy` state machine (see `/charly-internals:cloud-init-renderer`).
4. **Ensure guest ledger dir exists.** `ssh -- mkdir -p ~/.config/opencharly/installed/{deploys,layers}`.
5. **Resolve the guest home** once (`t.Exec.ResolveHome(ctx, "")`) and cache it on `t.guestHome`. Every home-bearing step field is resolved against THIS home, not the host operator's — see below.
6. **Walk plans.** Same batched `(Scope, Venue)` logic as `LocalDeployTarget`, but with `sudo bash -s` wrapped in `ssh`. See `/charly-local:local-deploy` for the grouping rules.
7. **Write the env.d-sourcing managed block** into the guest's detected login-shell init (`EnsureManagedBlockVia`) so the env.d files actually get sourced at login.

## Guest-home resolution (deploy-time `{{.Home}}`)

Home-bearing step fields — `ShellHookStep` env values + `path_append`,
`ShellSnippetStep` snippet/destination, `FileStep.Dest` — are compiled with the
deferred `{{.Home}}` token (`HomeToken`), NOT a baked compile-time home. Each
target resolves the token at emit via `InstallPlan.ResolveHome(home)`:
`img.Home` for OCI/pod-overlay, the host home for `LocalDeployTarget`, and the
**GUEST** home (`t.guestHome`) for `VmDeployTarget`. This is why a `target: vm`
deploy writes `/home/<guest-user>/.config/opencharly/env.d/<layer>.env` whose
contents point at `/home/<guest-user>/…` rather than the host operator's home.
`cmd:` task bodies are left untouched — `~`/`$HOME` there shell-expand at
runtime on the guest as the deploy user, already correct. See
`/charly-internals:install-plan` "Deferred home resolution".

## env.d-sourcing managed block (guest login shell)

`VmDeployTarget` calls `EnsureManagedBlockVia(ctx, t.Exec, shell, t.guestHome,
opts)` after the plan loop — the SAME executor-based writer `LocalDeployTarget`
uses (`shell_profile.go`; the os-based `EnsureManagedBlock` is a thin wrapper
over it). Without this block the per-layer env.d files exist but are never
sourced, so PATH never picks up `~/.npm-global/bin` etc. The shell is detected
from the GUEST `/etc/passwd` via `detectGuestShell` (getent), because the
guest's interactive default may differ from the operator's (CachyOS ships fish)
— writing bash syntax to `~/.profile` when the guest runs fish would never load.

## Cross-host builders (npm / pixi / cargo / aur)

`execBuilder` runs every builder on the HOST (podman) and ships the result into
the guest — guests never need a container runtime:

- **aur** → builds `.pkg.tar.zst` in a host staging dir, scp's them in, `pacman -U`.
- **npm / pixi / cargo** (`execHomeArtifactBuilder`) → bind-mounts a host staging
  dir AS the **guest home path** so npm shebangs / cargo rpaths / pixi activation
  scripts bake the path the guest will actually use, runs the same
  `renderBuilderScript` body as the local path, then tars the produced home
  subdirs (`~/.npm-global`, `~/.pixi`, `~/.cargo`; caches excluded), scp's the
  tarball in, and extracts it into the guest `$HOME` **as the guest user** so
  ownership + baked paths are correct. The builder image resolves via
  `resolveBuilderImage` (`--builder-image` → compiled `BuilderStep.BuilderImage`
  → `BuilderImageResolver`). Unknown builders honor `--skip-incompatible`.

This is what makes the full charly-cachyos stack — including the npm-builder AI CLIs
(`claude-code`, `codex`, `gemini`, `oracle`, `forgecode`) — install on a VM.

## RebootStep — the one step only this target executes

When a layer declares `reboot: true`, `BuildDeployPlan` appends a trailing
`RebootStep`. `VmDeployTarget.execReboot` is the sole executor that acts on it
(OCI/pod/k8s skip; `LocalDeployTarget` skips + warns — it never reboots the
operator host). It records the guest's `/proc/sys/kernel/random/boot_id`, fires
`(sleep 1; systemctl reboot) &` so the ssh session closes cleanly, then polls
until SSH answers AND the boot_id has changed — deterministic, not a fixed sleep,
so the still-up pre-reboot sshd can't be mistaken for "back up". This is what
lets a kernel-module layer (e.g. the CachyOS `nvidia-driver` layer) load its
module on a clean boot mid-deploy. See `/charly-internals:install-plan` RebootStep.

## Host→guest image transfer (`charly vm cp-box`)

`charly vm cp-box <vm> <ref> [--as <tag>] [--rootless]` (and the reusable
`TransferImageToGuest` helper) stream a host-built image into a running guest's
podman storage via `podman save | ssh podman load` (NO intermediate tarball —
the guest `/tmp` tmpfs is too small for a multi-GB image), idempotent (skips an
intact present image, re-streams a torn-overlay one — a name-only check would
wrongly skip a corrupt image) and offline (no registry). `--rootless` selects the
storage, and ALL of the load / integrity-probe / tag steps follow it consistently
(via the `podmanCmd(rootless)` helper):

- **default** → the guest's ROOT podman (`sudo podman`), for a `sudo podman run
  --device nvidia.com/gpu=all` consumer that needs `/dev/nvidia*` via root.
- **`--rootless`** → the SSH user's ROOTLESS podman (`podman`, no sudo; the tag
  runs via `RunUser`, not `RunSystem`). This is what `deployNestedPodsInGuest`
  uses: the nested pod comes up via the guest user's own `charly bundle from-box`
  (a `--user` quadlet) which reads the USER's storage, so the image MUST land
  there — a root-loaded image would be invisible to it.

## Nested pod-in-VM — persistent in-guest quadlet (`deployNestedPodsInGuest`)

A `target: vm` deploy whose `nested:` map has `target: pod` children brings each
child up as a PERSISTENT in-guest quadlet — the nested-pod-in-VM capability.
`VmUnifiedTarget.Add` constructs the `VmDeployTarget` and calls `deployNestedPodsInGuest` AFTER `VmDeployTarget.Emit` (so the guest's
own layers, including any kernel-driver reboot + the boot-time
`nvidia-ctk cdi generate`, are already applied). For each child it:

1. `charly box build <child.Image>` on the HOST (the guest needs no project).
2. `charly vm cp-box <vm> <child.Image> --as localhost/charly-<childKey>:latest
   --rootless` — into the guest USER's rootless podman.
3. over SSH as the guest user: `loginctl enable-linger` (so the `--user` quadlet
   auto-starts at boot and survives reboot), then `export
   XDG_RUNTIME_DIR=/run/user/$(id -u)` (so `systemctl --user` reaches the
   lingering user bus over the non-login SSH session — same requirement as
   VmDeployTarget's own user services), then the guest's own project-free
   `charly bundle from-box localhost/charly-<childKey>:latest <childKey>` — which
   generates + starts the quadlet from the image's baked OCI labels (ports,
   services, GPU device auto-detected in the guest; rootless GPU via CDI —
   `/dev/nvidia*` are world-rw and the CDI spec is world-readable).

Idempotent (cp-box skips an intact image; from-box re-applies on `charly update`).
The dispatch routes a VM-root deploy node-only (its pod children deploy in-guest
here, never via a host tree walk). `charly check live <vm>.<pod>` evaluates the
running nested pod by DELEGATING to the guest `charly check live <pod>` (where it is
a direct pod — guest-local podman + ports + the guest `charly`), so the protocol
verbs (cdp/wl/dbus/vnc/mcp) and `${HOST_PORT}` checks run natively instead of
skipping; see `/charly-check:check` "parent.child reaches the actual leaf". `charly vm
cp-box` is the host→guest image delivery for it.

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
    CharlyInstallStrategy       string                  // how charly is installed into the guest
    CloudInitRenderedDigest string                  // digest of the rendered cloud-init (re-render detection)
    Snapshots               []VmSnapshotState       // libvirt snapshot ledger
    Ephemeral               *EphemeralRuntime       // transient run-state for an ephemeral VM
}
```

Persisted in `~/.config/charly/charly.yml` as the `vm_state:` field on the VM's deploy entry (`BundleNode.VmState`). Each `charly vm build` / `charly vm create` / `charly bundle add vm:<name>` iteration updates the relevant fields. `charly bundle del vm:<name>` preserves the state (so re-adding picks up InstanceID etc.) unless `--purge` is passed.

## SSH key idempotency

`generateSSHKeypair` in `charly/vm.go` checks for `<vmStateDir>/id_ed25519.pub` before creating. Rebuilding a VM doesn't regenerate the keypair. First `charly vm build` writes the keypair; subsequent calls leave it untouched — so iterated rebuilds keep a stable pubkey and SSH stays valid.

## CLI dispatch: ResolveTarget → VmUnifiedTarget.Add / .Del

`charly bundle add vm:<name>` resolves via `deploy_add_cmd.go::dispatchNode` → `ResolveTarget` → `VmUnifiedTarget.Add` when the deploy name starts with `vm:` (or `target: vm` is set in `charly.yml`):

```
charly bundle add vm:arch ripgrep           # apply ripgrep layer in the guest
charly bundle add vm:arch fedora-coder \    # apply full fedora-coder layer set
    --add-candy team-extras \
    --add-candy github.com/team/configs/candy/sshkeys
charly bundle del vm:arch                   # reverse all applied layers in the guest
```

Prereq: VM must exist (`charly vm create arch` first). `VmUnifiedTarget.Add` does NOT auto-provision the VM — keeps the "provision" step explicit. If the VM is undefined, the dispatch returns a clean error pointing at `charly vm create`.

## passt backend + SSH port forwarding

When the VM's network uses libvirt user-mode + `<backend type='passt'/>` + `<portForward>` (see `/charly-internals:libvirt-renderer`), SSHExecutor connects to `127.0.0.1:<host-port>`. The portForward maps that through passt into the guest's `:22`. The indirection is invisible to SSHExecutor — it sees a normal TCP connect.

## Cross-References

- `/charly-internals:install-plan` — InstallPlan IR (the 4 DeployTarget implementers and the 9 step kinds)
- `/charly-internals:vm-spec` — VmSpec consumed by VmDeployTarget
- `/charly-internals:libvirt-renderer` — renders domain XML; portForward + passt backend
- `/charly-internals:cloud-init-renderer` — `EnsureCharlyInGuest` lives there
- `/charly-core:deploy` — `charly bundle add vm:<name>` command + charly.yml schema
- `/charly-local:local-deploy` — parallel target (LocalDeployTarget); ReverseOps model also used on VM target
- `/charly-vm:vm` — VM lifecycle; creates the target Emit runs against
- `/charly-vm:arch` — canonical worked example — VmDeployState persistence; ssh_key idempotency live-test
