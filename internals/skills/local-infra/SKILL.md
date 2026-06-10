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

The `InstallPlan` IR (see `/charly-internals:install-plan`) is the central data type. `LocalDeployTarget` (the executor) consumes the IR and invokes a handful of supporting files for each concrete concern: distro detection, ledger I/O, builder-container invocation, shell profile writes, ReverseOp execution, service rendering, ref resolution, and the managed ssh-config fragment that VMs publish on create.

## File map

| File | Purpose | Key exports |
|---|---|---|
| `charly/local_spec.go` | `LocalSpec` struct + `findLocalSpec` lookup | `LocalSpec`, `findLocalSpec` |
| `charly/deploy_target_local.go` | `LocalDeployTarget.Emit` IR consumer | `LocalDeployTarget` |
| `charly/unified_targets_local.go` | `LocalUnifiedTarget` adapter (Add/Del/Test/Update/Status/Shell/Rebuild) | `LocalUnifiedTarget` |
| `charly/ssh_managed_config.go` | `~/.config/charly/ssh_config` fragment writer (managed-block protocol) | `WriteVmSshStanza`, `RemoveVmSshStanza`, `ListVmSshAliases`, `EnsureSshConfigInclude`, `RemoveSshConfigInclude`, `VmSshAlias`, `SshFragmentPath`, `SshConfigPath` |
| `charly/deploy_executor_ssh.go` | Credential-free `SSHExecutor` (no `-i`, no host-key overrides) | `SSHExecutor` |
| `charly/deploy_executor.go` | `ShellExecutor` — local shell venue | `ShellExecutor` |
| `charly/hostdistro.go` | Detect host distro from `/etc/os-release`; glibc preflight | `HostDistro`, `DetectHostDistro`, `DetectHostGlibc`, `CompareGlibc`, `distroIDAliases` |
| `charly/install_ledger.go` | Flock-serialized JSON ledger at `~/.config/opencharly/installed/` | `LedgerPaths`, `LedgerLock`, `DeployRecord`, `CandyRecord`, `StepRecord`, `AcquireLedgerLock`, `AddCandyDeployment`, `RemoveCandyDeployment` |
| `charly/builder_run.go` | `podman run <builder>` wrapper for compile-needing layers | `BuilderRun`, `BuilderRunOpts`, `UserScopeBindMounts`, `UserScopeEnv` |
| `charly/shell_profile.go` | bash/zsh/fish detection + managed-block fencing + env.d I/O | `ShellKind`, `DetectLoginShell`, `EnvdDir`, `WriteEnvdFile`, `RemoveEnvdFile`, `EnsureManagedBlockVia`, `EnsureManagedBlock`, `RemoveManagedBlock`, `ShellInitFilePath` |
| `charly/reverse_ops.go` | Execute `ReverseOp` slices in LIFO order via per-kind handlers | `runReverseOps`, `ReverseExecutor` interface, 15 reverse handlers |

**Managed block is executor-based.** `EnsureManagedBlockVia(ctx, exec, shell, home, opts)` is the single writer for the env.d-sourcing block: `GetFile` the existing rc, merge the fenced block, `PutFile` it back. `LocalDeployTarget` calls it with a `ShellExecutor` (host fs); `VmDeployTarget` with an `SSHExecutor` (guest fs) — so the block lands in the right user's rc on local AND vm deploys (R3, one path). `EnsureManagedBlock(shell, home)` is a thin wrapper over it with a local `ShellExecutor`. `home` is the DESTINATION user's home — the guest home for a VM deploy (see `/charly-internals:vm-deploy-target`).
| `charly/service_render.go` | Render `ServiceEntry` → systemd unit / supervisord INI | `ServiceEntry`, `ServiceOverrides`, `RenderService`, `RenderedService` |
| `charly/deploy_ref.go` | Unified 4-form ref resolver | `DeployRef`, `RefKind`, `RefSource`, `ResolveDeployRef`, `classifyYAMLFile` |

## Credential-free SSHExecutor

`SSHExecutor` carries `User`, `Host`, `Port`, `Args`, `ConnectTimeout` — no `KeyPath`, no `KnownHostsFile`, no host-key overrides. `sshBaseArgs` emits only `-o LogLevel=ERROR` + `-o ConnectTimeout=…` plus optional `-p <port>` plus the deployment's pass-through `Args` plus the destination. `ssh(1)` reads `~/.ssh/config` + `ssh-agent` for everything else.

For VM destinations the Host field is the managed alias `charly-<vmname>`, written into `~/.config/charly/ssh_config` by `charly vm create`. Both `User` and `Port` are left empty — the alias's Host stanza supplies them.

## Managed ssh-config fragment

Pattern mirrors `charly/shell_profile.go` (the env.d managed block in `~/.profile` / `~/.zshenv` / `~/.config/fish/conf.d/opencharly.fish`):

- Fence markers: `# opencharly:begin (managed by charly; do not edit inside this block)` … `# opencharly:end`.
- `WriteVmSshStanza` writes/replaces a Host stanza inside the managed block at `~/.config/charly/ssh_config`. Multiple stanzas coexist, sorted by alias for deterministic output.
- `EnsureSshConfigInclude` ensures `Include ~/.config/charly/ssh_config` is present in `~/.ssh/config` (also fenced).
- `RemoveVmSshStanza` returns the count of remaining stanzas; when zero the fragment file is deleted.
- `RemoveSshConfigInclude` strips the Include line; when `~/.ssh/config` becomes empty after stripping, the file is deleted.
- `VmSshAlias("eval-arch-vm")` → `"charly-eval-arch-vm"`. Unique within the managed fragment.

## Ledger at `~/.config/opencharly/installed/`

Every target:local deploy writes a `DeployRecord` and per-layer `CandyRecord`s with `deployed_by:` refcount semantics. See the file map above for the JSON shapes.

## Cross-References

- `/charly-internals:install-plan` — the IR these files support; full step-kind catalogue.
- `/charly-internals:go` — overall Go code map; Kong framework; mode-purity invariant.
- `/charly-local:local-deploy` — user-facing target:local surface.
- `/charly-core:deploy` — command family overview.
- `/charly-image:layer` — unified `service:` schema authored by layer authors.

## When to Use This Skill

**MUST be invoked** when reading or modifying any of the listed files; when debugging target:local ledger state or ssh-config fragment behavior; when adding a new `ReverseOpKind` handler, a new shell to the detection map, or a new distro ID alias; or when extending the ref resolver to accept a new ref form.
