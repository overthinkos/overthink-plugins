---
name: gh
description: |
  GitHub CLI, git, and git-lfs — the single-responsibility home for all
  git/GitHub tooling. Ships the noscripts + post-install dance for git-lfs
  so the RPM's systemd trigger doesn't fail at build time.
  Use when composing git + gh + git-lfs into an image, or when deciding
  which layer should own a git-related binary.
---

# gh -- GitHub CLI + git + git-lfs

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages + one post-install task) |
| Depends | **(none)** |

## Packages

- RPM (with `--setopt=tsflags=noscripts`): `gh`, `git`, `git-lfs`
- pac: `github-cli`, `git`, `git-lfs`

## Why `tsflags=noscripts` + a post-install task

The `git-lfs` RPM's `%post` scriptlet runs `git-lfs install --system`
which tries to modify `/etc/` and talk to systemd — operations that
fail (loudly or silently) inside a buildah container. We install with
noscripts and then run the git-lfs hook configuration manually:

```yaml
tasks:
  - cmd: /usr/bin/git-lfs install --system --skip-repo 2>/dev/null || true
    user: root
```

The `|| true` tolerates distros/versions where the command layout
differs; `--skip-repo` prevents git-lfs from trying to touch a repo
that doesn't exist in the build container.

## Single-responsibility ownership

This layer is the exclusive home for `gh`, `git`, and `git-lfs` — no other
layer (including `/ov-coder:dev-tools`) installs them. That keeps ownership
unambiguous ("which layer do I look at to update the git-lfs version?" — this
one) and avoids duplicate test ids (`gh-binary` collisions).

Effect for layer authors: any image that wants git tooling composes `gh`
explicitly. The four power-user images (`arch-ov`, `fedora-ov`,
`fedora-coder`, `githubrunner` via the `ov-full` chain) all list `gh`
explicitly.

## Tests

Six build-scope tests:

| Test | Purpose |
|---|---|
| `gh-binary` | `/usr/bin/gh` exists |
| `gh-version` | `gh --version` exits 0 |
| `git-binary` | `/usr/bin/git` exists |
| `git-version` | `git --version` exits 0 |
| `git-lfs-binary` | `/usr/bin/git-lfs` exists |
| `git-lfs-version` | `git-lfs --version` exits 0 |

## Cross-distro coverage

`rpm:` (Fedora — from the `github-cli` COPR / community repo), `pac:` (Arch — `github-cli` from `extra`), `deb:` (Debian/Ubuntu — adds `https://cli.github.com/packages` as an apt repo with signed-by key; ships `gh`, `git`, `git-lfs`). Full parity across all three package families.

## Usage

```yaml
# image.yml or layer.yml
layers:
  - gh
```

## Used In Images

- `/ov-coder:arch-ov`, `/ov-distros:fedora-ov`, `/ov-coder:fedora-coder` — power-user images that compose gh explicitly
- `/ov-openclaw:openclaw-desktop` — streaming-desktop sibling
- Any image composing `hermes-full`

## Related Layers

- `/ov-coder:dev-tools` — does not install git/gh/git-lfs (this layer owns them)
- `/ov-distros:agent-forwarding` — pairs with gh for SSH/GPG agent access (you usually want both when driving gh from inside a container with the host's GPG keys forwarded)
- `/ov-distros:github-runner` — self-hosted Actions runner; different layer, different purpose
- `/ov-coder:github-actions` — installs `act` + `actionlint` for local Actions testing; also different from this layer

## Related Commands

- `/ov-build:secrets` — provision `GITHUB_TOKEN` for `gh auth login`
- `/ov-core:shell` — run gh interactively inside a container

## When to Use This Skill

**MUST be invoked** when:

- Adding git or GitHub CLI access to an image (compose this layer —
  DO NOT add `gh`, `git`, or `git-lfs` to any other layer's packages).
- Debugging why `git-lfs install` fails at build time (the noscripts +
  post-install pattern here is the fix).
- Understanding why `/ov-coder:dev-tools` does not install gh
  (this layer holds single-responsibility ownership of git tooling).

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
