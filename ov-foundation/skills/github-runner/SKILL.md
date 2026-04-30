---
name: github-runner
description: |
  GitHub Actions self-hosted runner as a supervised container service.
  Use when working with GitHub Actions runners, CI/CD infrastructure, or runner registration.
---

# github-runner -- GitHub Actions self-hosted runner

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord` |
| Volumes | `state` -> `/opt/actions-runner`, `storage` -> `/var/lib/containers/storage` |
| Service | `github-runner` (supervisord) |
| Security | `privileged: true` |
| Install files | `tasks:` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `RUNNER_ALLOW_RUNASROOT` | `1` |
| `RUNNER_WORK_DIR` | `/opt/actions-runner/_work` |
| `RUNNER_GROUP` | `Default` |
| `OV_BUILD_ENGINE` | `podman` |
| `OV_RUN_ENGINE` | `podman` |

## Packages

- `skopeo`, `jq`, `libcap`, `podman`, `buildah`, `golang`, `git` (RPM)
- `openssh-clients`, `systemd-container`, `qemu-user-static-aarch64` (RPM)

## Lifecycle Hooks

- **post_enable** -- Registers runner with `RUNNER_ORG` and `RUNNER_TOKEN`
- **pre_remove** -- Deregisters runner (requires `RUNNER_TOKEN`)

## Usage

```yaml
# image.yml
githubrunner:
  layers:
    - github-runner
```

```bash
ov config githubrunner -e RUNNER_ORG=myorg -e RUNNER_TOKEN=xxx
ov remove githubrunner -e RUNNER_TOKEN=xxx
```

## Used In Images

- `/ov-foundation:githubrunner`

## Related Layers

- `/ov-foundation:supervisord` -- process manager dependency

## When to Use This Skill

Use when the user asks about:

- GitHub Actions self-hosted runners
- Runner registration or deregistration
- CI/CD container infrastructure
- `RUNNER_TOKEN` or `RUNNER_ORG` configuration

## Author + Test References

- `/ov-build:layer` — layer authoring reference (tasks, vars, env_provides, tests block syntax)
- `/ov-build:eval` — declarative testing framework for the `tests:` block
