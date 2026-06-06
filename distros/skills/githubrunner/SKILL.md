---
name: githubrunner
description: |
  Self-hosted GitHub Actions runner on CachyOS, fully rootless — runs as
  uid=1000 with zero added capabilities (no root, no privileged) via
  container-nesting, with rootless nested podman/buildah/skopeo for CI.
  Host networking retained for reachability. MUST be invoked before building,
  deploying, configuring, or troubleshooting the githubrunner image.
---

# githubrunner

Self-hosted GitHub Actions runner on **CachyOS**, fully rootless. Shares the
rootless nested-container posture with `/ov-openclaw:openclaw-desktop` (uid=1000,
no caps, `unmask=/proc/*` via `/ov-distros:container-nesting`).

## Image Properties

| Property | Value |
|----------|-------|
| Base | cachyos (`cachyos.cachyos`) |
| Layers | agent-forwarding, github-runner, ov, dbus, container-nesting |
| Build | pac |
| Platforms | linux/amd64 |
| UID / user | **1000 / user** (rootless — NO uid/privileged override) |
| Network | host (reach the host-side ghcr pull-through mirror) |
| Security | container-nesting's posture: `cap_add:[]`, `security_opt:[unmask=/proc/*]`, devices `/dev/fuse` + `/dev/net/tun` |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `cachyos` (`docker.io/cachyos/cachyos-v3`, via the `cachyos` import namespace)
2. `agent-forwarding` — GPG/SSH/direnv (composes gnupg + direnv + ssh-client)
3. `github-runner` — the Actions runner agent (under `${HOME}/actions-runner`),
   cosign, go/git/jq, qemu-user-static (aarch64 cross-arch CI), the .NET runtime
   deps, the credential-backed registration, the ghcr mirror config
4. `ov` — the ov binary + `virtualization` + `gocryptfs` + `socat`
5. `dbus` — session bus (for runner hooks)
6. `container-nesting` — rootless nested podman/buildah/skopeo + the subuid/subgid
   layout + newuidmap/newgidmap file-caps + containers/storage/policy configs

## Rootless posture (genuinely uid 1000, no caps)

The image carries **no uid/user/privileged override**, so it resolves to
`container-nesting`'s posture: uid=1000, `cap_add:[]`, `security_opt:[unmask=/proc/*]`,
devices `/dev/fuse` + `/dev/net/tun`. The runner process (`run.sh`), all CI jobs,
and rootless nested podman all run as `user` (uid 1000). See
`/ov-distros:container-nesting` for the `mount_too_revealing()` kernel RCA that
makes nested podman work without caps or `--privileged`.

`/ov-coder:sshd` (passwordless `/etc/sudoers.d/ov-user`) is composable if a
workflow genuinely needs sudo; it is not composed by default.

## Nested container support

`githubrunner` composes `/ov-distros:container-nesting` directly, so CI jobs get
first-class **rootless nested podman/buildah/skopeo** (uid 1000, `/dev/fuse` +
`unmask=/proc/*`, `crun` + `fuse-overlayfs`, `BUILDAH_ISOLATION=chroot`). The
rootless image cache lives under `${HOME}/.local/share/containers`. Cross-arch
container builds (aarch64) work via the repo `qemu-user-static` +
`qemu-user-static-binfmt` packages.

## Token mechanism (credential-backed; obtained via `gh`)

`RUNNER_TOKEN` is a `secret_accept` (credential-store-backed — never written to
deploy.yml or the quadlet); `RUNNER_ORG` is an `env_accept`. The registration
token is short-lived and only consumed once at `ov config` time (the registered
runner persists its own `.credentials` on the `state` volume), so it is obtained
fresh from `gh`. The `post_enable`/`pre_remove` hooks are guarded — they skip
when `RUNNER_TOKEN` is empty, so a token-less deploy (e.g. an eval bed) brings the
image up without registering, and a stale token never errors a teardown.

```bash
# Obtain a fresh org registration token and register:
TOKEN=$(gh api -X POST /orgs/<org>/actions/runners/registration-token --jq .token)
ov config githubrunner -e RUNNER_ORG=<org> -e RUNNER_TOKEN="$TOKEN"   # token scrubbed → credential store
ov start githubrunner
```

Removal deregisters via the `pre_remove` hook, which needs a **remove**-token
(distinct from the registration token):

```bash
ov remove githubrunner -e RUNNER_TOKEN=$(gh api -X POST /orgs/<org>/actions/runners/remove-token --jq .token)
```

## Lifecycle

```bash
ov box build githubrunner
ov config githubrunner -e RUNNER_ORG=<org> -e RUNNER_TOKEN="$TOKEN"
ov start githubrunner
ov stop githubrunner
ov remove githubrunner -e RUNNER_TOKEN=<remove-token>
```

## Key Layers

- `/ov-distros:github-runner` — runner agent, hooks, registration, ghcr mirror, .NET deps
- `/ov-distros:container-nesting` — rootless nested podman/buildah/skopeo, subuid layout, caps
- `/ov-tools:ov` — the ov binary + virtualization + gocryptfs + socat
- `/ov-distros:agent-forwarding` — GPG/SSH/direnv for the `.secrets` workflow

## Related Images

- `/ov-distros:cachyos` — the CachyOS base (parent, via the `cachyos` namespace)
- `/ov-openclaw:openclaw-desktop` — same rootless container-nesting posture (uid 1000, no caps)
- `/ov-distros:fedora-ov`, `/ov-coder:arch-ov` — the ov-toolchain siblings (root path, image-level full-hammer security)

## Related Commands

- `/ov-core:ov-config` — deploy setup with RUNNER_ORG (env) / RUNNER_TOKEN (secret)
- `/ov-core:start`, `/ov-core:stop`, `/ov-core:remove` — lifecycle + pre-remove deregistration
- `/ov-core:ov-status`, `/ov-core:logs` — verify runner is idle + troubleshoot
- `/ov-build:secrets` — the credential store backing `RUNNER_TOKEN`

## Verification

Build-scope (`ov eval box githubrunner`) + deploy-scope (`ov eval live`) checks
ship on the `github-runner` layer (functional: `config.sh --version` proves the
.NET deps resolved; rootless nested `podman run` proves the posture). The
disposable R10 bed is **`eval-githubrunner-pod`** (`ov eval run eval-githubrunner-pod`)
— it proves the rootless composition WITHOUT GitHub registration (no token →
guarded hooks no-op).

After `ov start` against a registered deploy:

- `ov status githubrunner` — container running
- `ov shell githubrunner -c "id"` — uid=1000(user) (rootless)
- `ov shell githubrunner -c "podman run --rm quay.io/libpod/alpine:latest true"` — rootless nested podman works
- Runner appears as "Idle" in the org's Settings > Actions > Runners

## When to Use This Skill

**MUST be invoked** when the task involves the githubrunner image,
self-hosted runners, or GitHub Actions CI/CD. Invoke this skill BEFORE
reading source code or launching Explore agents.

## Related

- `/ov-image:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
- `/ov-eval:eval` — the `eval:` checks + the `eval-githubrunner-pod` R10 bed
