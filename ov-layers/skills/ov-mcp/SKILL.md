---
name: ov-mcp
description: |
  MCP server exposing the full ov CLI as tools (Streamable HTTP on port
  18765). Meta-layer composition — layers: [ov, supervisord] — ships only
  service wiring + `/workspace` bind-mount + OV_PROJECT_DIR env plumbing.
  Auto-falls back to the upstream overthinkos/overthink repo when /workspace
  has no image.yml. Use when composing an MCP gateway into any image so LLM
  agents can drive ov remotely.
---

# ov-mcp -- MCP server deployment layer

## Layer Properties

| Property | Value |
|----------|-------|
| Kind | Meta-layer (no install files of its own) |
| Composition | `layers: [ov, supervisord]` |
| Port | 18765 (Streamable HTTP MCP endpoint at `/mcp`) |
| Service | supervisord-managed `ov-mcp` program |
| Volumes | `project` → `/workspace` (bind-mount the project root from the host) |
| Env | `OV_PROJECT_DIR: "/workspace"` |
| mcp_provides | `{name: ov, url: http://{{.ContainerName}}:18765/mcp, transport: http}` |

**Volume naming note:** the volume NAME stays `project` (deployer-facing
API: `ov config <image> --bind project=/path`) but the in-container PATH
is `/workspace`. Renamed from `/project` → `/workspace` in 2026-04 for a
more neutral term that works regardless of whether the bind mount is an
overthink checkout or any other dev workspace.

## What It Provides

Deploys `ov mcp serve --listen :18765` inside the container under
supervisord. The server exposes the entire ov CLI (auto-generated from
Kong reflection, currently ~192 tools including the 2026 authoring
surface — project scaffolding, YAML editing, file-write verbs) as MCP
over Streamable HTTP. Any image composing `ov-mcp` advertises itself
via the `org.overthinkos.mcp_provides` OCI label, so consumers —
Claude Code, Open WebUI, OpenClaw, or the in-repo `ov eval mcp` client
— can drive it without any out-of-band URL configuration.

See `/ov:mcp` Part 2 for the full server architecture: Kong
reflection, destructive-hint annotations, `--read-only` filter,
transport dispatch.

## Composition vs. dependency

This layer uses `layers:`, not `depends:` — deliberately. The
distinction:

- `depends:` says "my install needs these layers installed first."
- `layers:` says "I *am* these layers plus my additions."

`ov-mcp` installs no packages and copies no files — it's pure wiring
(service block, mcp_provides declaration, volumes/env, one mkdir
task to create `/workspace` with 0777 so `ov version` works even
when no bind-mount is attached). A meta-layer composition (`layers:`)
captures that exactly. The validator requires every layer to ship
*something* installable; `layers:` satisfies that by transitively
pulling in the children's install files.

## Three deployment patterns

Build-mode MCP tools (`image.build`, `image.list.images`,
`image.inspect`, etc.) need to read `image.yml`. The ov-mcp layer
supports three paths, in order of "how much local setup":

**1. Bind-mount a local project** (maximal local iteration):

```bash
ov config <image> --bind project=/home/you/overthink
ov start <image>
```

The agent reads image.yml + layers/ directly from the host — any
local edit is immediately visible.

**2. Pin a remote repo** (reproducible, no local checkout):

```bash
ov config <image> -e OV_PROJECT_REPO=overthinkos/overthink@<sha>
ov start <image>
```

At serve-startup, `ov mcp serve` clones into
`~/.cache/ov/repos/github.com/overthinkos/overthink@<sha>` and chdirs
there. No bind mount needed. Good for CI, headless dev, or shipping
an agent that always drives a specific upstream version.

**3. Auto-fallback** (zero setup — the 2026-04 default):

```bash
ov config <image>
ov start <image>
# Nothing bound to /workspace; /workspace is world-writable but empty.
# ov mcp serve's bootstrapProject() detects no image.yml in cwd and
# silently clones + chdirs into the default overthinkos/overthink
# cache. Logs a single line naming the reason.
```

Opt out with `--no-default-repo` (hard-fail instead of falling back).
The top-level ov CLI never auto-fetches — only `ov mcp serve` does.

**Subtle 2026-04 change:** previously the layer set `OV_PROJECT_DIR=/workspace`
in the container env and `bootstrapProject()` early-returned on that env
being set. Now `bootstrapProject()` ignores `OV_PROJECT_DIR` by itself
and checks for an actual `image.yml` in the resolved cwd — falling
back to the default repo if missing. This matters because
`OV_PROJECT_DIR=/workspace` is permanently set by this layer's `env:`
block, so the old early-return meant the auto-fallback was dead code
for any ov-mcp container. The new logic makes pattern 3 genuinely work
by default. See `/ov:mcp` "Project-dir wiring" for the full RCA.

## Tests

Six deploy-scope tests ship with the layer:

| Test | Purpose |
|---|---|
| `ov-mcp-service` | supervisord `ov-mcp` program is running |
| `ov-mcp-port` | host `127.0.0.1:${HOST_PORT:18765}` reachable |
| `mcp-ov-ping` | MCP `ping` succeeds over the in-repo client (URL rewritten via `rewriteMCPURLForHost`, host-networked containers included) |
| `mcp-ov-list-tools` | MCP `list-tools` returns a catalog containing the canonical `image.build`, `status`, `test.mcp.ping` entries |
| `mcp-ov-call-version` | MCP `call version` returns the in-container CalVer (proves round-trip of a safe tool) |
| `mcp-ov-call-list-images` | MCP `call image.list.images` returns images — **proves the bind-mount OR auto-fallback is working** (matches "fedora" either way, since upstream overthinkos/overthink always has a `fedora` image) |

All `mcp:` checks pass `mcp_name: ov` so they stay unambiguous on
images that also expose `jupyter` or `chrome-devtools` servers
(e.g. `/ov-images:selkies-desktop-ov`).

## Host networking caveat (resolved 2026-04)

Previously `rewriteMCPURLForHost` hard-failed on host-networked
containers because their `NetworkSettings.Ports` is empty. The
`ov/mcp_client.go` `lookupHostPort()` function now detects
`HostConfig.NetworkMode == "host"` and returns the container port
verbatim (container ports ARE host ports under `network: host`). See
`ov/testvars.go` `IsHostNetworked()` + the mirror fix in
`mergeRuntimeVars()` for `HOST_PORT:<N>` env-var population.

Practical impact: ov-mcp now works on both bridge-networked images
(e.g. `/ov-images:arch-ov`) and host-networked ones (e.g.
`/ov-images:fedora-coder`, `/ov-images:fedora-ov`).

## Port choice rationale

Default `:18765` chosen for non-collision with sibling MCP layers:

- `8888` — jupyter-mcp
- `9224` — chrome-devtools-mcp (via mcp-proxy)
- `18789` — openclaw gateway

## Used In Images

Compose `ov-mcp` into any image that should be reachable as an MCP
gateway. Current users:

- `/ov-images:fedora-coder` — kitchen-sink dev image; uses pattern 3 (auto-fallback) by default, pattern 1 when the developer wants local edits visible.
- `/ov-images:arch-ov` — Arch-based ov toolchain image (bridge network, ports 2222/18765).

Images composing `ov-mcp` must publish port 18765 (either via
layer-declared `ports: [18765]` that auto-collects into the
container's EXPOSE, or an image-level `ports: ["18765:18765"]` block
in `image.yml`). Both `network: host` and the default ov bridge work.

## Related Layers

- `/ov-layers:ov` — The underlying binary layer this wraps.
- `/ov-layers:supervisord` — Init system for the `ov-mcp` program.
- `/ov-layers:jupyter-mcp` — Sibling MCP server (notebook manipulation, FastMCP-based).
- `/ov-layers:chrome-devtools-mcp` — Sibling MCP server (browser automation, mcp-proxy wrapper).

## Cross-References

- `/ov:mcp` — **Part 2: Server** is the authoritative reference for `ov mcp serve` architecture, destructive-hint policy, `--read-only` filter, capture model, and the new bootstrapProject() logic.
- `/ov:image` — "Project directory resolution" covers the `-C` / `--dir` / `OV_PROJECT_DIR` global flag and `--repo` / `OV_PROJECT_REPO`.
- `/ov:config` — `--bind project=<path>` is the deployer's handshake with this layer's `volumes:` declaration.
- `/ov:eval` — Deploy-scope `mcp:` test verb methods used here.
- `/ov-dev:go` — `ov/mcp_server.go` `bootstrapProject()` implementation, including the env-var proxy detection of top-level flags and the unconditional image.yml check (new in 2026-04).

## When to Use This Skill

**MUST be invoked** when:

- Adding `ov-mcp` to an image's layer list.
- Debugging why a build-mode MCP tool returns stale data or an unexpected
  image list (is the agent reading the bind-mount or the auto-fallback?).
- Authoring an ov-like CLI's MCP deployment and wanting the reference
  pattern (`layers:` composition + volumes + env + service + auto-fallback).
- Investigating port-18765 collisions or MCP URL rewriting on composed
  images (especially host-networked ones).

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
