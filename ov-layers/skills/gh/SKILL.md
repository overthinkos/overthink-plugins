---
name: gh
description: |
  GitHub CLI, git, and git-lfs — the single-responsibility home for all
  git/GitHub tooling as of 2026-04. Ships the noscripts + post-install
  dance for git-lfs so the RPM's systemd trigger doesn't fail at build time.
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
fail (loudly or silently) inside a buildah container. Per the same
pattern `dev-tools` used to carry, we install with noscripts and then
run the git-lfs hook configuration manually:

```yaml
tasks:
  - cmd: /usr/bin/git-lfs install --system --skip-repo 2>/dev/null || true
    user: root
```

The `|| true` tolerates distros/versions where the command layout
differs; `--skip-repo` prevents git-lfs from trying to touch a repo
that doesn't exist in the build container.

## Single-responsibility split (2026-04)

Previously `/ov-layers:dev-tools` ALSO installed `gh` and `git-lfs`
— two layers with overlapping responsibility, duplicate test ids
(`gh-binary` collision), and unclear ownership ("which layer do I
look at to update the git-lfs version?"). In 2026-04 the git tooling
was moved exclusively here; dev-tools dropped `gh`, `git-lfs`, and
the git-lfs post-install task.

Effect for layer authors: anywhere an image previously got git
tooling via `dev-tools`, it now needs to compose `gh` explicitly.
The four power-user images (`arch-ov`, `fedora-ov`, `fedora-coder`,
`githubrunner` via the `ov-full` chain) already list `gh`
explicitly so they were unaffected.

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

- `/ov-images:arch-ov`, `/ov-images:fedora-ov`, `/ov-images:fedora-coder` — power-user images that compose gh explicitly
- `/ov-images:selkies-desktop-ov` — streaming-desktop sibling
- Any image composing `hermes-full`

## Related Layers

- `/ov-layers:dev-tools` — no longer installs git/gh/git-lfs (2026-04 split)
- `/ov-layers:agent-forwarding` — pairs with gh for SSH/GPG agent access (you usually want both when driving gh from inside a container with the host's GPG keys forwarded)
- `/ov-layers:github-runner` — self-hosted Actions runner; different layer, different purpose
- `/ov-layers:github-actions` — installs `act` + `actionlint` for local Actions testing; also different from this layer

## Related Commands

- `/ov:secrets` — provision `GITHUB_TOKEN` for `gh auth login`
- `/ov:shell` — run gh interactively inside a container

## When to Use This Skill

**MUST be invoked** when:

- Adding git or GitHub CLI access to an image (compose this layer —
  DO NOT add `gh`, `git`, or `git-lfs` to any other layer's packages).
- Debugging why `git-lfs install` fails at build time (the noscripts +
  post-install pattern here is the fix).
- Understanding why `/ov-layers:dev-tools` no longer installs gh
  (the 2026-04 single-responsibility split lives here).

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
