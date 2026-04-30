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
Shares the rootless-first posture with `/ov-foundation:fedora-ov`,
`/ov-coder:arch-ov`, and `/ov-coder:fedora-coder`.

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
posture was dropped once the `/ov-foundation:container-nesting` kernel
RCA proved surgical `unmask=/proc/*` works without caps. See
`/ov-foundation:fedora-ov` for the full rationale and
`/ov-foundation:container-nesting` for the `mount_too_revealing()` RCA.

The `/ov-foundation:github-runner` layer and runner-hook scripts still
invoke `sudo` where they genuinely need root (e.g. for system-level
docker setup). Passwordless sudo is provided indirectly — if you
need it for your workflows, compose `/ov-coder:sshd` into the
layer list (it installs `/etc/sudoers.d/ov-user`).

## Nested container support

Unlike `/ov-foundation:fedora-ov` and `/ov-coder:arch-ov`, `githubrunner`
does NOT compose `/ov-foundation:container-nesting` directly. Nested
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

- `/ov-foundation:github-runner` — runner agent, hooks, security config
- `/ov-coder:ov-full` — ov + virtualization + gocryptfs + socat
- `/ov-foundation:virtualization` — QEMU/KVM + rootless libvirt session daemon
- `/ov-foundation:container-nesting` — not composed here by default (add if workflows need first-class rootless nested containers)

## Related Images

- `/ov-foundation:fedora` — parent base
- `/ov-foundation:fedora-ov` — sibling uid=1000 ov toolchain (includes `container-nesting` directly)
- `/ov-coder:arch-ov` — Arch counterpart of fedora-ov
- `/ov-coder:fedora-coder` — kitchen-sink dev image sharing the same security posture
- `/ov-selkies:selkies-desktop-ov` — non-desktop alternative streaming-desktop with the same rootless posture

## Related Commands

- `/ov-core:config` — deploy setup with RUNNER_ORG / RUNNER_TOKEN
- `/ov-core:start`, `/ov-core:stop`, `/ov-core:remove` — lifecycle + pre-remove deregistration
- `/ov-core:status`, `/ov-core:logs` — verify runner is idle + troubleshoot

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

## Related

- `/ov-build:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
