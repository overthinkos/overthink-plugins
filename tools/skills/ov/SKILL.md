---
name: ov
description: |
  Overthink CLI (ov) binary installed into container/VM images for in-container use.
  Use when working with ov binary deployment inside containers, native D-Bus support, or the full ov toolchain (ov binary + virtualization + gocryptfs + socat).
---

# ov -- Overthink CLI binary

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `task:`, `bin/ov` |

## What It Provides

The `ov` binary inside containers serves two purposes:

1. **Native D-Bus agent** — `ov eval dbus` commands on the host delegate to an in-venue `ov` via `engine exec container ov eval dbus <cmd> . <args>`, which connects to the local D-Bus session bus using `godbus/dbus/v5` (pure Go, no external tools needed). The primary path for `ov eval dbus notify`, `ov eval dbus call`, `ov eval dbus list`, and `ov eval dbus introspect`. **The image need NOT bake the `ov` layer for this:** when the venue lacks `ov`, the explicit dbus commands COPY the host's own binary in on demand (the generic copy-`ov`-into-a-running-venue mechanism, `EnsureOvInVenue` over `DeployExecutor.PutFile` — `podman cp` for a container, `scp` for a VM/host) and invoke the delivered copy. Baking the layer only pre-stages the binary so the first call skips the copy.

2. **In-container CLI** — full ov functionality available inside the container for scripting, service management, and automation.

## Updating the Binary — dual-path gotcha

The `ov` layer's `copy: bin/ov` task is resolved **relative to the layer directory**, so the image build reads `candy/ov/bin/ov` — NOT the repo-root `bin/ov`. Two independent paths need to stay in sync:

| Path | Who reads it |
|------|-------------|
| `bin/ov` (repo root) | Host-side `ov` invocations; users running `/tmp/ov` style tests |
| `candy/ov/bin/ov` | The `ov` layer's COPY into images during `ov box build` |

**Canonical workflow** — `task build:ov` compiles to repo-root AND syncs to the layer:

```bash
task build:ov                              # Builds + syncs both paths; rebuild images after.
ov box build <image>                     # Rebuild affected images.
```

**Manual workflow** — if you skip `task build:ov` and build with `go build` directly, you MUST sync the layer path, or images will bake the previous binary:

```bash
cd ov && go build -o ../bin/ov .           # Only updates repo-root bin/ov.
cp bin/ov candy/ov/bin/ov                 # REQUIRED — sync to layer path.
ov box build <image>                     # Rebuild affected images.
```

**Why this bites**: `ov box build` uses auto-generated intermediate images (e.g., `ghcr.io/overthinkos/fedora-ov-2-dbus-nodejs`) that cache the `ov` layer. If you update `bin/ov` in repo-root but forget the layer copy, the intermediate's cache hit serves stale content. After cleaning up a stale dual-path situation, `podman rmi 'ghcr.io/overthinkos/fedora-ov-2*'` forces a clean intermediate rebuild.

## `ov status` Probe

The `ov` probe checks:
1. Whether the `ov` binary exists in the container
2. The CalVer version (`ov version`)

Shows as `ov:ok (2026.94.1417)` in `ov status` detail view. Returns `-` for images without the `ov` layer.

**Note:** `ov version` writes to **stdout** via `fmt.Println` (the prior
`println(version)` emitted to stderr; the move to `fmt.Println` landed
with the MCP server work so the in-process tool-call path — which
captures `os.Stdout` — returns the CalVer correctly). The layer test at
`candy/ov/candy.yml` asserts `stdout:` matches `[0-9]{4}\.[0-9]+`. The
`ov status` probe uses `CombinedOutput()` so it's agnostic to the
stream.

## Usage

```yaml
# box.yml -- now included in all images with supervisord
my-image:
  layers:
    - ov
```

## Used In Images

- The `ov` layer is the full toolchain (ov binary + virtualization + gocryptfs + socat); composed into githubrunner, fedora-ov, arch-ov
- Now directly added to all images with supervisord (openclaw, jupyter, ollama, sway-browser-vnc, selkies-desktop, immich, etc.)

## Related Layers

- `/ov-infrastructure:virtualization`, `/ov-infrastructure:gocryptfs`, `/ov-infrastructure:socat` -- the layers the `ov` layer composes alongside the binary to form the full toolchain
- `/ov-coder:ov-mcp` -- layers: [ov, supervisord] meta-composition that deploys `ov mcp serve` (~192-tool MCP gateway) with a `/workspace` bind mount (volume NAME `project`) for build-mode tools + auto-fallback to overthinkos/overthink when nothing is bound
- `/ov-infrastructure:dbus-layer` -- D-Bus session bus (ov eval dbus commands need this)
- `/ov-selkies:swaync` -- notification daemon (needed for ov eval dbus notify to show popups)

## When to Use This Skill

Use when the user asks about:

- Installing the ov binary inside containers
- The full ov toolchain composition (ov binary + virtualization + gocryptfs + socat)
- In-container ov CLI usage
- Native D-Bus support (ov eval dbus commands delegate to in-container binary)
- Updating the ov candy binary after code changes

## Related

- `/ov-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval box`, `ov eval live`)
