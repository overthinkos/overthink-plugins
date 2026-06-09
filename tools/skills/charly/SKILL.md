---
name: charly
description: |
  OpenCharly CLI (charly) binary installed into container/VM images for in-container use.
  Use when working with charly binary deployment inside containers, native D-Bus support, or the full charly toolchain (charly binary + virtualization + gocryptfs + socat).
---

# charly -- OpenCharly CLI binary

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `task:`, `bin/charly` |

## What It Provides

The `charly` binary inside containers serves two purposes:

1. **Native D-Bus agent** — `charly eval dbus` commands on the host delegate to an in-venue `charly` via `engine exec container charly eval dbus <cmd> . <args>`, which connects to the local D-Bus session bus using `godbus/dbus/v5` (pure Go, no external tools needed). The primary path for `charly eval dbus notify`, `charly eval dbus call`, `charly eval dbus list`, and `charly eval dbus introspect`. **The image need NOT bake the `charly` layer for this:** when the venue lacks `charly`, the explicit dbus commands COPY the host's own binary in on demand (the generic copy-`charly`-into-a-running-venue mechanism, `EnsureCharlyInVenue` over `DeployExecutor.PutFile` — `podman cp` for a container, `scp` for a VM/host) and invoke the delivered copy. Baking the layer only pre-stages the binary so the first call skips the copy.

2. **In-container CLI** — full charly functionality available inside the container for scripting, service management, and automation.

## Updating the Binary — dual-path gotcha

The `charly` layer's `copy: bin/charly` task is resolved **relative to the layer directory**, so the image build reads `candy/charly/bin/charly` — NOT the repo-root `bin/charly`. Two independent paths need to stay in sync:

| Path | Who reads it |
|------|-------------|
| `bin/charly` (repo root) | Host-side `charly` invocations; users running `/tmp/charly` style tests |
| `candy/charly/bin/charly` | The `charly` layer's COPY into images during `charly box build` |

**Canonical workflow** — `task build:charly` compiles to repo-root AND syncs to the layer:

```bash
task build:charly                              # Builds + syncs both paths; rebuild images after.
charly box build <image>                     # Rebuild affected images.
```

**Manual workflow** — if you skip `task build:charly` and build with `go build` directly, you MUST sync the layer path, or images will bake the previous binary:

```bash
cd charly && go build -o ../bin/charly .           # Only updates repo-root bin/charly.
cp bin/charly candy/charly/bin/charly                 # REQUIRED — sync to layer path.
charly box build <image>                     # Rebuild affected images.
```

**Why this bites**: `charly box build` uses auto-generated intermediate images (e.g., `ghcr.io/overthinkos/charly-fedora-2-dbus-nodejs`) that cache the `charly` layer. If you update `bin/charly` in repo-root but forget the layer copy, the intermediate's cache hit serves stale content. After cleaning up a stale dual-path situation, `podman rmi 'ghcr.io/overthinkos/charly-fedora-2*'` forces a clean intermediate rebuild.

## `charly status` Probe

The `charly` probe checks:
1. Whether the `charly` binary exists in the container
2. The CalVer version (`charly version`)

Shows as `charly:ok (2026.94.1417)` in `charly status` detail view. Returns `-` for images without the `charly` layer.

**Note:** `charly version` writes to **stdout** via `fmt.Println` (the prior
`println(version)` emitted to stderr; the move to `fmt.Println` landed
with the MCP server work so the in-process tool-call path — which
captures `os.Stdout` — returns the CalVer correctly). The layer test at
`candy/charly/candy.yml` asserts `stdout:` matches `[0-9]{4}\.[0-9]+`. The
`charly status` probe uses `CombinedOutput()` so it's agnostic to the
stream.

## Usage

```yaml
# box.yml -- now included in all images with supervisord
my-image:
  layers:
    - charly
```

## Used In Images

- The `charly` layer is the full toolchain (charly binary + virtualization + gocryptfs + socat); composed into githubrunner, charly-fedora, charly-arch
- Now directly added to all images with supervisord (openclaw, jupyter, ollama, sway-browser-vnc, selkies-desktop, immich, etc.)

## Related Layers

- `/charly-infrastructure:virtualization`, `/charly-infrastructure:gocryptfs`, `/charly-infrastructure:socat` -- the layers the `charly` layer composes alongside the binary to form the full toolchain
- `/charly-coder:charly-mcp` -- layers: [charly, supervisord] meta-composition that deploys `charly mcp serve` (~192-tool MCP gateway) with a `/workspace` bind mount (volume NAME `project`) for build-mode tools + auto-fallback to overthinkos/overthink when nothing is bound
- `/charly-infrastructure:dbus-layer` -- D-Bus session bus (charly eval dbus commands need this)
- `/charly-selkies:swaync` -- notification daemon (needed for charly eval dbus notify to show popups)

## When to Use This Skill

Use when the user asks about:

- Installing the charly binary inside containers
- The full charly toolchain composition (charly binary + virtualization + gocryptfs + socat)
- In-container charly CLI usage
- Native D-Bus support (charly eval dbus commands delegate to in-container binary)
- Updating the charly candy binary after code changes

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
