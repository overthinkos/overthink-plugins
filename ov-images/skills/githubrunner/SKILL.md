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
| Layers | github-runner, ov-full |
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

Runs as root with host networking. Required for nested container builds and VM operations in CI.

## Registration

```bash
ov enable githubrunner -e RUNNER_ORG=myorg -e RUNNER_TOKEN=<token>
# post_enable hook runs config.sh --unattended
```

Removal deregisters via `pre_remove` hook:

```bash
ov remove githubrunner -e RUNNER_TOKEN=<token>
```

## Lifecycle

```bash
ov build githubrunner
ov enable githubrunner -e RUNNER_ORG=myorg -e RUNNER_TOKEN=<token>
ov start githubrunner
ov stop githubrunner
ov remove githubrunner -e RUNNER_TOKEN=<token>
```

## Key Layers

- `/ov-layers:github-runner` — runner agent, hooks, security config
- `/ov-layers:ov-full` — ov + virtualization + gocryptfs + socat

## Related Images

- `/ov-images:fedora` — parent base

## Verification

After `ov start`:
- `ov status githubrunner` — container running
- `ov service status githubrunner` — all supervisord services RUNNING
- Check runner appears as "Idle" in GitHub org/repo Settings > Actions > Runners

## When to Use This Skill

**MUST be invoked** when the task involves the githubrunner image, self-hosted runners, or GitHub Actions CI/CD. Invoke this skill BEFORE reading source code or launching Explore agents.
