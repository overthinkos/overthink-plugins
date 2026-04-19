---
name: githubrunner
description: |
  Self-hosted GitHub Actions runner with full ov toolchain.
  Runs as root with host networking, includes VMs, crypto, and container tools.
  MUST be invoked before building, deploying, configuring, or troubleshooting the githubrunner image.
---

# githubrunner

Self-hosted GitHub Actions runner with the full Overthink toolchain.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, github-runner, ov-full |
| Platforms | linux/amd64 |
| UID/GID | 0/0 (root) |
| User | root |
| Network | host |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (quay.io/fedora/fedora:43)
2. `pixi` → `python` → `supervisord` (transitive)
3. `github-runner` — Actions runner agent, skopeo, podman, buildah
4. `ov-full` — ov CLI + virtualization + gocryptfs + socat

## Security

Runs as root with host networking. The image asserts its posture at the
**image level** in `image.yml`:

```yaml
githubrunner:
  security:
    cap_add: [ALL]
    security_opt:
      - label=disable
      - seccomp=unconfined
```

`githubrunner` does NOT compose `/ov-layers:container-nesting` directly
(only transitively via `ov-full`'s composition chain, and in this
image's case the layer list is `[agent-forwarding, github-runner,
ov-full, dbus]` — no explicit `container-nesting`). So the resolved OCI
security label is just `cap_add:[ALL] + security_opt:[label=disable,
seccomp=unconfined]` — no `unmask=/proc/*` (unlike `/ov-images:fedora-ov`
and `/ov-images:arch-ov` which include `container-nesting` directly and
therefore inherit `unmask=/proc/*` + `/dev/fuse` + `/dev/net/tun` from
that layer).

If the GitHub Actions runner needs nested rootless podman, add
`container-nesting` to the layer list. Otherwise the image-level
`cap_add:[ALL]` + host networking is sufficient for most CI workloads
that spawn containers via `docker`/`podman` against the host socket.

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

## Related Images

- `/ov-images:fedora` — parent base
- `/ov-images:fedora-ov` — sibling root-mode ov toolchain (includes `container-nesting` directly)
- `/ov-images:arch-ov` — Arch counterpart of fedora-ov
- `/ov-images:selkies-desktop-ov` — non-root ov toolchain wrapped in a streaming desktop

## Related Layers

- `/ov-layers:github-runner` — runner agent package + hooks
- `/ov-layers:ov-full` — ov + virtualization (including virtqemud supervisord program) + gocryptfs + socat
- `/ov-layers:virtualization` — QEMU/KVM + rootless libvirt session daemon
- `/ov-layers:container-nesting` — nested rootless podman recipe (not composed here by default — add if your workflows need it)

## Verification

After `ov start`:
- `ov status githubrunner` — container running
- `ov service status githubrunner` — all services RUNNING
- Check runner appears as "Idle" in GitHub org/repo Settings > Actions > Runners

## When to Use This Skill

**MUST be invoked** when the task involves the githubrunner image, self-hosted runners, or GitHub Actions CI/CD. Invoke this skill BEFORE reading source code or launching Explore agents.
