---
name: charly
description: |
  OpenCharly CLI (charly) binary installed into container/VM images for in-container use.
  Use when working with charly binary deployment inside containers, native D-Bus support, or the full charly toolchain (charly binary + virtualization + gocryptfs + socat).
---

# charly -- OpenCharly CLI binary

## Candy Properties

| Property | Value |
|----------|-------|
| Install files | `run:` step, `bin/charly` |

## What It Provides

The `charly` binary inside containers serves two purposes:

1. **Native D-Bus agent** — `charly check dbus` commands on the host delegate to an in-venue `charly` via `engine exec container charly check dbus <cmd> . <args>`, which connects to the local D-Bus session bus using `godbus/dbus/v5` (pure Go, no external tools needed). The primary path for `charly check dbus notify`, `charly check dbus call`, `charly check dbus list`, and `charly check dbus introspect`. **The box need NOT bake the `charly` candy for this:** when the venue lacks `charly`, the explicit dbus commands COPY the host's own binary in on demand (the generic copy-`charly`-into-a-running-venue mechanism, `EnsureCharlyInVenue` over `DeployExecutor.PutFile` — `podman cp` for a container, `scp` for a VM/host) and invoke the delivered copy. Baking the candy only pre-stages the binary so the first call skips the copy.

2. **In-container CLI** — full charly functionality available inside the container for scripting, service management, and automation.

## Check-vs-production binary source — disposable beds bake the IN-DEV charly

The `charly` candy installs the binary as a proper, dependency-resolving OS
package via `localpkg:` (`{pac: pkg/arch, rpm: pkg/fedora, deb: pkg/debian}`).
The BINARY SOURCE depends on the box type — a hard distinction, NEVER mixed:

- **Disposable check beds** (`disposable: true` bundles) bake the latest **in-development**
  charly: the check-bed runner builds every bed image with `charly box build
  --dev-local-pkg`, so the package is BUILT from the local working tree
  (`pkg/<fmt>` + `charly/`). A bed therefore tests the charly code under
  development — never a stale published release.
- **Production boxes** bake the latest **published** charly: a normal
  `charly box build` DOWNLOADS the published release package
  (`releases/latest/download/opencharly-<arch>.<fmt>`).

ONE decision point (`renderLocalPkgImageInstall`), generic across all kinds and
all localpkg candies; the check-bed runner sets `--dev-local-pkg` automatically, a
production build never does. A dev build that cannot find its local source HARD
errors (R4 — no silent fallback to the release). Full mechanics:
`/charly-internals:install-plan` "Check-vs-production charly toolchain". This is
WHY a fresh check bed exercises your uncommitted charly changes while a real box
ships the released toolchain.

## Updating the Binary — dual-path gotcha

The `charly` candy's `copy: bin/charly` run step is resolved **relative to the candy directory**, so the box build reads `candy/charly/bin/charly` — NOT the repo-root `bin/charly`. Two independent paths need to stay in sync:

| Path | Who reads it |
|------|-------------|
| `bin/charly` (repo root) | Host-side `charly` invocations; users running `/tmp/charly` style tests |
| `candy/charly/bin/charly` | The `charly` candy's COPY into boxes during `charly box build` |

**Canonical workflow** — `task build:charly` compiles to repo-root AND syncs to the layer:

```bash
task build:charly                              # Builds + syncs both paths; rebuild images after.
charly box build <image>                     # Rebuild affected images.
```

**Manual workflow** — if you skip `task build:charly` and build with `go build` directly, you MUST sync the candy path, or boxes will bake the previous binary:

```bash
cd charly && go build -o ../bin/charly .           # Only updates repo-root bin/charly.
cp bin/charly candy/charly/bin/charly                 # REQUIRED — sync to layer path.
charly box build <image>                     # Rebuild affected images.
```

**Why this bites**: `charly box build` uses auto-generated intermediate images (e.g., `ghcr.io/overthinkos/charly-fedora-2-dbus-nodejs`) that cache the `charly` candy. If you update `bin/charly` in repo-root but forget the candy copy, the intermediate's cache hit serves stale content. After cleaning up a stale dual-path situation, `charly clean --invalidate 'charly-fedora-2*'` forces a clean intermediate rebuild.

## `charly status` Probe

The `charly` probe checks:
1. Whether the `charly` binary exists in the container
2. The CalVer version (`charly version`)

Shows as `charly:ok (2026.94.1417)` in `charly status` detail view. Returns `-` for boxes without the `charly` candy.

**Note:** `charly version` writes to **stdout** via `fmt.Println` (the prior
`println(version)` emitted to stderr; the move to `fmt.Println` landed
with the MCP server work so the in-process tool-call path — which
captures `os.Stdout` — returns the CalVer correctly). The candy test at
`candy/charly/charly.yml` asserts `stdout:` matches `[0-9]{4}\.[0-9]+`. The
`charly status` probe uses `CombinedOutput()` so it's agnostic to the
stream.

## Usage

```yaml
# charly.yml — name-first; the charly candy is now composed into all supervisord images
my-image:
  box:
    base: fedora
  my-image-candy:
    candy:
      - charly
```

## Used In Boxes

- The `charly` candy is the full toolchain (charly binary + virtualization + gocryptfs + socat); composed into githubrunner, charly-fedora, charly-arch
- Now directly added to all boxes with supervisord (openclaw, jupyter, ollama, sway-browser-vnc, selkies-desktop, immich, etc.)

## Related Candies

- `/charly-infrastructure:virtualization`, `/charly-infrastructure:gocryptfs`, `/charly-infrastructure:socat` -- the candies the `charly` candy composes alongside the binary to form the full toolchain
- `/charly-coder:charly-mcp` -- candies: [charly, supervisord] meta-composition that deploys `charly mcp serve` (~192-tool MCP gateway) with a `/workspace` bind mount (volume NAME `project`) for build-mode tools + auto-fallback to overthinkos/overthink when nothing is bound
- `/charly-infrastructure:dbus-layer` -- D-Bus session bus (charly check dbus commands need this)
- `/charly-selkies:swaync` -- notification daemon (needed for charly check dbus notify to show popups)

## When to Use This Skill

Use when the user asks about:

- Installing the charly binary inside containers
- The full charly toolchain composition (charly binary + virtualization + gocryptfs + socat)
- In-container charly CLI usage
- Native D-Bus support (charly check dbus commands delegate to in-container binary)
- Updating the charly candy binary after code changes

## Related

- `/charly-image:layer` — candy authoring reference (`charly.yml` schema, plan-step verbs, service declarations)
- `/charly-check:check` — declarative testing (`check:` block, `charly check box`, `charly check live`)
