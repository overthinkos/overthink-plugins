---
name: local-infra
description: |
  Go file map for the target:local execution surface. Files: local_spec.go,
  deploy_target_local.go, unified_targets_local.go, ssh_managed_config.go,
  hostdistro.go, install_ledger.go, builder_run.go, shell_profile.go,
  reverse_ops.go, service_render.go, deploy_ref.go.
  MUST be invoked before reading or modifying any of those files, or when
  debugging target:local deploy behaviour (ledger state, sudo batching,
  managed-block insertion, glibc preflight, ssh-config fragment, ref resolution).
---

# Local Infra — Go files that support target:local deployments

## Overview

The `InstallPlan` IR (see `/ov-dev:install-plan`) is the central data type. `LocalDeployTarget` (the executor) consumes the IR and invokes a handful of supporting files for each concrete concern: distro detection, ledger I/O, builder-container invocation, shell profile writes, ReverseOp execution, service rendering, ref resolution, and the managed ssh-config fragment that VMs publish on create.

Renamed from `host-infra` in the local-cutover. `HostDeployTarget` → `LocalDeployTarget`, `HostUnifiedTarget` → `LocalUnifiedTarget`, `LocalDeployExecutor` → `ShellExecutor`, `findHostSpec` → `findLocalSpec` (relocated from `k8s_cmd.go` to `local_spec.go`).

## File map

| File | Purpose | Key exports |
|---|---|---|
| `ov/local_spec.go` | `LocalSpec` struct + `findLocalSpec` lookup | `LocalSpec`, `findLocalSpec` |
| `ov/deploy_target_local.go` | `LocalDeployTarget.Emit` IR consumer | `LocalDeployTarget` |
| `ov/unified_targets_local.go` | `LocalUnifiedTarget` adapter (Add/Del/Test/Update/Status/Shell/Rebuild) | `LocalUnifiedTarget` |
| `ov/ssh_managed_config.go` | `~/.config/ov/ssh_config` fragment writer (managed-block protocol) | `WriteVmSshStanza`, `RemoveVmSshStanza`, `ListVmSshAliases`, `EnsureSshConfigInclude`, `RemoveSshConfigInclude`, `VmSshAlias`, `SshFragmentPath`, `SshConfigPath` |
| `ov/deploy_executor_ssh.go` | Credential-free `SSHExecutor` (no `-i`, no host-key overrides) | `SSHExecutor` |
| `ov/deploy_executor.go` | `ShellExecutor` (was `LocalDeployExecutor`) — local shell venue | `ShellExecutor` |
| `ov/hostdistro.go` | Detect host distro from `/etc/os-release`; glibc preflight | `HostDistro`, `DetectHostDistro`, `DetectHostGlibc`, `CompareGlibc`, `distroIDAliases` |
| `ov/install_ledger.go` | Flock-serialized JSON ledger at `~/.config/overthink/installed/` | `LedgerPaths`, `LedgerLock`, `DeployRecord`, `LayerRecord`, `StepRecord`, `AcquireLedgerLock`, `AddLayerDeployment`, `RemoveLayerDeployment` |
| `ov/builder_run.go` | `podman run <builder>` wrapper for compile-needing layers | `BuilderRun`, `BuilderRunOpts`, `UserScopeBindMounts`, `UserScopeEnv` |
| `ov/shell_profile.go` | bash/zsh/fish detection + managed-block fencing + env.d I/O | `ShellKind`, `DetectLoginShell`, `EnvdDir`, `WriteEnvdFile`, `RemoveEnvdFile`, `EnsureManagedBlock`, `RemoveManagedBlock`, `ShellInitFilePath` |
| `ov/reverse_ops.go` | Execute `ReverseOp` slices in LIFO order via per-kind handlers | `runReverseOps`, `ReverseExecutor` interface, 15 reverse handlers |
| `ov/service_render.go` | Render `ServiceEntry` → systemd unit / supervisord INI | `ServiceEntry`, `ServiceOverrides`, `RenderService`, `RenderedService` |
| `ov/deploy_ref.go` | Unified 4-form ref resolver | `DeployRef`, `RefKind`, `RefSource`, `ResolveDeployRef`, `classifyYAMLFile` |

## Credential-free SSHExecutor

`SSHExecutor` carries `User`, `Host`, `Port`, `Args`, `ConnectTimeout` — no `KeyPath`, no `KnownHostsFile`, no host-key overrides. `sshBaseArgs` emits only `-o LogLevel=ERROR` + `-o ConnectTimeout=…` plus optional `-p <port>` plus the deployment's pass-through `Args` plus the destination. `ssh(1)` reads `~/.ssh/config` + `ssh-agent` for everything else.

For VM destinations the Host field is the managed alias `ov-<vmname>`, written into `~/.config/ov/ssh_config` by `ov vm create`. Both `User` and `Port` are left empty — the alias's Host stanza supplies them.

## Managed ssh-config fragment

Pattern mirrors `ov/shell_profile.go` (the env.d managed block in `~/.profile` / `~/.zshenv` / `~/.config/fish/conf.d/overthink.fish`):

- Fence markers: `# overthink:begin (managed by ov; do not edit inside this block)` … `# overthink:end`.
- `WriteVmSshStanza` writes/replaces a Host stanza inside the managed block at `~/.config/ov/ssh_config`. Multiple stanzas coexist, sorted by alias for deterministic output.
- `EnsureSshConfigInclude` ensures `Include ~/.config/ov/ssh_config` is present in `~/.ssh/config` (also fenced).
- `RemoveVmSshStanza` returns the count of remaining stanzas; when zero the fragment file is deleted.
- `RemoveSshConfigInclude` strips the Include line; when `~/.ssh/config` becomes empty after stripping, the file is deleted.
- `VmSshAlias("arch-vm")` → `"ov-arch-vm"`. Unique within the managed fragment.

## Ledger at `~/.config/overthink/installed/`

Unchanged from host-infra — every target:local deploy writes a `DeployRecord` and per-layer `LayerRecord`s with `deployed_by:` refcount semantics. See file map above; the JSON shapes did not change.

## Cross-References

- `/ov-dev:install-plan` — the IR these files support; full step-kind catalogue.
- `/ov-dev:go` — overall Go code map; Kong framework; mode-purity invariant.
- `/ov-advanced:local-deploy` — user-facing target:local surface.
- `/ov-core:deploy` — command family overview.
- `/ov-build:layer` — unified `services:` schema authored by layer authors.

## When to Use This Skill

**MUST be invoked** when reading or modifying any of the listed files; when debugging target:local ledger state or ssh-config fragment behavior; when adding a new `ReverseOpKind` handler, a new shell to the detection map, or a new distro ID alias; or when extending the ref resolver to accept a new ref form.
