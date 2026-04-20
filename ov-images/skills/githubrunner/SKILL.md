---
name: githubrunner
description: |
  Self-hosted GitHub Actions runner with the full Overthink toolchain.
  Rootless-first since 2026-04 — runs as uid=1000 with passwordless sudo
  (no root, no cap_add: ALL). Host networking retained for reachability.
  MUST be invoked before building, deploying, configuring, or troubleshooting
  the githubrunner image.
---

# githubrunner

Self-hosted GitHub Actions runner with the full Overthink toolchain.
Shares the rootless-first posture with `/ov-images:fedora-ov`,
`/ov-images:arch-ov`, and `/ov-images:fedora-coder`.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, github-runner, ov-full, dbus |
| Platforms | linux/amd64 |
| UID / user | **1000 / user** (rootless-first since 2026-04) |
| Network | host |
| Security | layer-level only |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (quay.io/fedora/fedora:43)
2. Transitive (via `ov-full`): `ov` + `virtualization` + `gocryptfs` + `socat`
3. `github-runner` — Actions runner agent, skopeo, podman, buildah
4. `dbus` — session bus (for runner hooks)

## Rootless-first posture (2026-04 refactor)

Previously ran as `uid: 0 / user: root` with `cap_add: [ALL]` +
`security_opt: [label=disable, seccomp=unconfined]`. That legacy
posture was dropped once the `/ov-layers:container-nesting` kernel
RCA proved surgical `unmask=/proc/*` works without caps. See
`/ov-images:fedora-ov` for the full rationale and
`/ov-layers:container-nesting` for the `mount_too_revealing()` RCA.

The `/ov-layers:github-runner` layer and runner-hook scripts still
invoke `sudo` where they genuinely need root (e.g. for system-level
docker setup). Passwordless sudo is provided indirectly — if you
need it for your workflows, compose `/ov-layers:sshd` into the
layer list (it installs `/etc/sudoers.d/ov-user`).

## Nested container support

Unlike `/ov-images:fedora-ov` and `/ov-images:arch-ov`, `githubrunner`
does NOT compose `/ov-layers:container-nesting` directly. Nested
rootless podman works only via `ov-full`'s transitive dependency
chain. If your Actions workflows need first-class rootless nested
containers (with `/dev/fuse` + `unmask=/proc/*` security opts), add
`container-nesting` to the layer list explicitly.

For most CI workloads that spawn containers via docker/podman against
the host socket (host networking covers it), the current composition
is enough.

## Registration

```bash
ov config githubrunner -e RUNNER_ORG=myorg -e RUNNER_TOKEN=<token>
# post_enable hook runs config.sh --unattended
```

Removal deregisters via `pre_remove` hook:

```bash
ov remove githubrunner -e RUNNER_TOKEN=<token>
```

## Lifecycle

```bash
ov image build githubrunner
ov config githubrunner -e RUNNER_ORG=myorg -e RUNNER_TOKEN=<token>
ov start githubrunner
ov stop githubrunner
ov remove githubrunner -e RUNNER_TOKEN=<token>
```

## Key Layers

- `/ov-layers:github-runner` — runner agent, hooks, security config
- `/ov-layers:ov-full` — ov + virtualization + gocryptfs + socat
- `/ov-layers:virtualization` — QEMU/KVM + rootless libvirt session daemon
- `/ov-layers:container-nesting` — not composed here by default (add if workflows need first-class rootless nested containers)

## Related Images

- `/ov-images:fedora` — parent base
- `/ov-images:fedora-ov` — sibling uid=1000 ov toolchain (includes `container-nesting` directly)
- `/ov-images:arch-ov` — Arch counterpart of fedora-ov
- `/ov-images:fedora-coder` — kitchen-sink dev image sharing the same security posture
- `/ov-images:selkies-desktop-ov` — non-desktop alternative streaming-desktop with the same rootless posture

## Related Commands

- `/ov:config` — deploy setup with RUNNER_ORG / RUNNER_TOKEN
- `/ov:start`, `/ov:stop`, `/ov:remove` — lifecycle + pre-remove deregistration
- `/ov:status`, `/ov:logs` — verify runner is idle + troubleshoot

## Verification

After `ov start`:

- `ov status githubrunner` — container running
- `ov service status githubrunner` — all services RUNNING
- `ov shell githubrunner -c "id"` — uid=1000(user)
- Check runner appears as "Idle" in GitHub org/repo Settings > Actions > Runners

## When to Use This Skill

**MUST be invoked** when the task involves the githubrunner image,
self-hosted runners, or GitHub Actions CI/CD. Invoke this skill BEFORE
reading source code or launching Explore agents.
