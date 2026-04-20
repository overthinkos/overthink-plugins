---
name: ov-mcp
description: |
  MCP server exposing the full ov CLI as tools (Streamable HTTP on port 18765). Meta-layer composition — layers: [ov, supervisord] — ships no install of its own, only service wiring + project-dir bind-mount + OV_PROJECT_DIR env plumbing. Use when composing an MCP gateway into any image so LLM agents can drive ov remotely.
---

# ov-mcp -- MCP server deployment layer

## Layer Properties

| Property | Value |
|----------|-------|
| Kind | Meta-layer (no install files of its own) |
| Composition | `layers: [ov, supervisord]` |
| Port | 18765 (Streamable HTTP MCP endpoint at `/mcp`) |
| Service | supervisord-managed `ov-mcp` program |
| Volumes | `project` → `/project` (bind the project root from the host) |
| Env | `OV_PROJECT_DIR: "/project"` |
| mcp_provides | `{name: ov, url: http://{{.ContainerName}}:18765/mcp, transport: http}` |

## What It Provides

Deploys `ov mcp serve --listen :18765` inside the container under supervisord. The server exposes the entire ov CLI (176 tools) as MCP over Streamable HTTP. Any image that composes `ov-mcp` advertises itself as an MCP server via the `org.overthinkos.mcp_provides` OCI label, so consumers — Claude Code, Open WebUI, OpenClaw, or the in-repo `ov test mcp` client — can drive it without any out-of-band URL configuration.

See `/ov:mcp` (Part 2) for the full server architecture: Kong reflection, destructive-hint annotations, `--read-only` filter, transport dispatch.

## Composition vs. dependency

This layer uses `layers:`, not `depends:` — deliberately. The distinction:

- `depends:` says "my install needs these layers installed first."
- `layers:` says "I *am* these layers plus my additions."

`ov-mcp` installs no packages and copies no files — it's pure wiring (service block, mcp_provides declaration, volumes/env). A meta-layer composition (`layers:`) captures that exactly. The validator requires every layer to ship *something* installable; `layers:` satisfies that by transitively pulling in the children's install files.

## Wiring a project directory

Build-mode MCP tools (`image.build`, `image.list.images`, `image.inspect`, etc.) need to read `image.yml`. Inside a container the cwd is `/root` where no project exists. The `volumes` + `env` pair closes that gap:

```yaml
volumes:
  - name: project
    path: /project

env:
  OV_PROJECT_DIR: "/project"
```

At config time, bind-mount the host project into the container:

```bash
ov config arch-ov --bind project=/home/you/overthink
ov start arch-ov
```

The ov binary's global `-C` / `--dir` / `OV_PROJECT_DIR` flag (`ov/main.go`) calls `os.Chdir(OV_PROJECT_DIR)` before Kong dispatches the subcommand — so every `os.Getwd()` call in build-mode code paths picks up `/project` transparently, without any per-command plumbing. See `/ov:image` "Project directory resolution" for the flag and `/ov:config` "Bind-mounting a project checkout for ov mcp serve" for the config workflow.

If the deployer forgets the `--bind`, build-mode tools return a clear error (`reading image.yml: open /project/image.yml: no such file or directory`) but session-wide tools (`status`, `version`, `doctor`, `secrets`, `settings`) continue to work.

## Tests

Five deploy-scope tests ship with the layer:

| Test | Purpose |
|---|---|
| `ov-mcp-service` | supervisord `ov-mcp` program is running |
| `ov-mcp-port` | host 127.0.0.1:${HOST_PORT:18765} reachable |
| `mcp-ov-ping` | MCP `ping` succeeds over the in-repo client (URL rewritten via `rewriteMCPURLForHost`) |
| `mcp-ov-list-tools` | MCP `list-tools` returns a catalog containing the canonical `image.build`, `status`, `test.mcp.ping` entries |
| `mcp-ov-call-version` | MCP `call version` returns the in-container CalVer (proves round-trip of a safe tool) |
| `mcp-ov-call-list-images` | MCP `call image.list.images` returns the project's images — **proves the bind-mount + `OV_PROJECT_DIR` wiring** end-to-end |

All `mcp:` checks pass `mcp_name: ov` so they stay unambiguous on images that also expose `jupyter` or `chrome-devtools` servers (e.g. `selkies-desktop-ov`).

## Port choice rationale

Default `:18765` chosen for non-collision with sibling MCP layers:
- `8888` — jupyter-mcp
- `9224` — chrome-devtools-mcp (via mcp-proxy)
- `18789` — openclaw gateway

## Used In Images

Compose `ov-mcp` into any image that should be reachable as an MCP gateway. Current users:

- `/ov-images:arch-ov` — Arch-based full-ov-toolchain image (bridge network, ports 2222/18765).
- Future: `/ov-images:fedora-ov` should follow the same pattern when migrating off `network: host` to the default ov bridge.

Images composing `ov-mcp` must publish port 18765 (either via layer-declared `ports: [18765]` that auto-collects into the container's EXPOSE, or an image-level `ports: ["18765:18765"]` block in `image.yml`). `network: host` works too but breaks `rewriteMCPURLForHost` because host-networked containers have no published-port mappings — the bridge-network + explicit port publishing pattern is portable.

## Related Layers

- `/ov-layers:ov` — The underlying binary layer this wraps.
- `/ov-layers:supervisord` — Init system for the `ov-mcp` program.
- `/ov-layers:jupyter-mcp` — Sibling MCP server (notebook manipulation, FastMCP-based).
- `/ov-layers:chrome-devtools-mcp` — Sibling MCP server (browser automation, mcp-proxy wrapper).

## Cross-References

- `/ov:mcp` — **Part 2: Server** is the full authoritative reference for `ov mcp serve` architecture, destructive-hint policy, `--read-only` filter, capture model, and deployment.
- `/ov:image` — "Project directory resolution" covers the `-C` / `--dir` / `OV_PROJECT_DIR` global flag that this layer's `env:` block targets.
- `/ov:config` — `--bind project=<path>` is the deployer's handshake with this layer's `volumes:` declaration.
- `/ov:test` — Deploy-scope `mcp:` test verb methods used here.
- `/ov-dev:go` — `ov/mcp_server.go` implementation map (Kong reflection, capture model, destructive set).

## When to Use This Skill

**MUST be invoked** when:

- Adding `ov-mcp` to an image's layer list.
- Debugging why a build-mode MCP tool (`image.list.images`, etc.) returns an `image.yml not found` error (check the bind mount and `OV_PROJECT_DIR`).
- Authoring an ov-like CLI's own MCP deployment and wanting the reference pattern (`layers:` composition + volumes + env + service).
- Investigating port-18765 collisions or MCP URL rewriting on composed images.
