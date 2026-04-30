---
name: mcp
description: |
  MUST be invoked before any work involving: Model Context Protocol — both directions. (1) `ov eval mcp` client: probing MCP servers declared via mcp_provides, testing MCP tool catalogs, debugging the URL-rewriter (including host-networked containers via `HostConfig.NetworkMode` detection — new 2026-04) or port-publishing behavior. (2) `ov mcp serve` server: running the ov CLI itself as an MCP server over Streamable HTTP or stdio, auto-generated from Kong reflection (~192 tools including the MCP-first authoring surface — image/layer scaffolding, comment-preserving YAML edits, free-form file writes), destructive-hint annotations, the `--read-only` filter, auto-fallback to `overthinkos/overthink` when cwd has no `image.yml` (always fires now, regardless of OV_PROJECT_DIR being set — 2026-04 change), and the `ov-mcp` deployment layer with its `/workspace` bind mount.
---

# MCP - Model Context Protocol (client + server)

`ov` speaks MCP in both directions. This skill covers both:

- **Client** — `ov eval mcp <method>`: connect to any MCP server declared via `mcp_provides` and probe/call/read. Used to test MCP endpoints shipped by `jupyter-mcp`, `chrome-devtools-mcp`, or the ov server itself.
- **Server** — `ov mcp serve`: expose the entire ov CLI surface (build + test + deploy modes, 190 tools) as MCP over Streamable HTTP or stdio. Used by LLM agents (Claude Code, Open WebUI, OpenClaw) to drive ov remotely. Deployed in-container via the `ov-mcp` layer. The 190-tool catalog now includes a project-scaffolding + YAML-editing + file-write authoring surface, so an agent can build an `ov` project from scratch over RPC — see "Authoring tools" below.

Both surfaces share the same SDK: `github.com/modelcontextprotocol/go-sdk v1.5.0`.

---

# Part 1 — Client (`ov eval mcp …`)

## Overview

`ov eval mcp` connects to Model Context Protocol servers declared by running containers via `mcp_provides`, using [github.com/modelcontextprotocol/go-sdk](https://pkg.go.dev/github.com/modelcontextprotocol/go-sdk) (v1.5.0). Seven leaf subcommands cover the full MCP client surface: `ping`, `servers`, `list-tools`, `list-resources`, `list-prompts`, `call`, `read`. No MCP URL argument is ever typed by the user — the verb reads the target image's `org.overthinkos.mcp_provides` OCI label, resolves `{{.ContainerName}}` templates, applies pod-aware `localhost` rewriting, and maps the container-network URL to the published host port automatically.

### Also as a declarative verb

Every `ov eval mcp <method>` is authorable as an `mcp:` verb inside a `tests:` block. The method name becomes the verb's YAML value; method-specific args are sibling fields (`tool:`, `uri:`, `input:`, `mcp_name:`). Shared matchers (`stdout:`, `stderr:`, `exit_status:`, `timeout:`) work like other verbs. See `/ov-build:eval` for the parent router and the complete method allowlist. Example:

```yaml
- mcp: list-tools
  scope: deploy
  stdout:
    - contains: insert_cell
    - contains: execute_cell
```

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Ping | `ov eval mcp ping <image>` | Liveness check — returns `ok` on a successful `Ping` RPC |
| Enumerate | `ov eval mcp servers <image>` | List MCP servers declared by the image (no dial) |
| List tools | `ov eval mcp list-tools <image>` | Tool name + first-line description per line |
| List resources | `ov eval mcp list-resources <image>` | URI + name + MIME type per line |
| List prompts | `ov eval mcp list-prompts <image>` | Prompt name + description per line |
| Call | `ov eval mcp call <image> <tool> [<input-json>]` | Invoke a tool; prints TextContent payload |
| Read | `ov eval mcp read <image> <uri>` | Read a resource; prints Text content |

Every leaf accepts:
- `-i <instance>` — target a specific instance
- `--name <server>` — disambiguate when the image declares multiple `mcp_provides` entries
- `--json` — emit the SDK's native result struct instead of plaintext
- `--timeout <duration>` — per-operation timeout (default 30s)

## Architecture

The verb runs entirely on the host — no delegation into the container — and builds a single `*mcp.ClientSession` per invocation.

1. **Container resolution**: `resolveContainer(image, instance)` → `ov-<image>[-<instance>]`.
2. **Image ref + metadata**: `containerImageRef` → `ExtractMetadata` → `meta.MCPProvides` (read from OCI label `org.overthinkos.mcp_provides`).
3. **Template substitution**: any `{{.ContainerName}}` in the URL is replaced with the resolved container name.
4. **Pod-aware rewrite**: `podAwareMCPProvides` folds same-image entries so the URL host becomes `localhost` (identical to the OV_MCP_SERVERS path used by `ov config`).
5. **Host-side port rewrite**: the load-bearing piece. `rewriteMCPURLForHost` parses the URL; if the host is the container name or `localhost`, it looks up the published host port for the URL's port via `podman inspect` (same `NetworkSettings.Ports` data that powers `${HOST_PORT:N}` in declarative tests) and rewrites to `127.0.0.1:<host-port>`. Non-matching hosts (external URLs) pass through unchanged.
6. **Transport pick**: `transport: http` (or empty) → `StreamableClientTransport{Endpoint}`; `transport: sse` → `SSEClientTransport{Endpoint}`; any other string errors at dial time with the declared transport echoed.
7. **Session**: `mcp.NewClient(&mcp.Implementation{Name: "ov", Version: ComputeCalVer()}, nil).Connect(ctx, transport, nil)` — runs the MCP `initialize` handshake and returns an opened `ClientSession`. A `defer session.Close()` always fires.

Source: `ov/mcp.go` (Kong subcommand tree + leaf Run() methods), `ov/mcp_client.go` (resolver, URL rewriter, transport picker, formatters). The URL rewriter is the one truly novel piece; the rest composes existing infrastructure. See `/ov-dev:go` "Declarative Testing" section for the implementation map.

## Commands

### Ping

```bash
ov eval mcp ping jupyter
# ok
```

Calls `ClientSession.Ping(ctx, nil)`. Exit 0 iff the server responds. Useful as a liveness check, often paired with a short `--timeout` in CI.

### Servers (discovery only, no dial)

```bash
ov eval mcp servers jupyter
# jupyter   http://localhost:8888/mcp   http

ov eval mcp servers jupyter --json
# [{"name":"jupyter","url":"http://localhost:8888/mcp","transport":"http","source":"jupyter"}]
```

Reads `mcp_provides` from the OCI label, applies template substitution + pod-aware rewrite, prints the result. No MCP handshake. Useful for confirming which server names the image advertises before dialing.

### List tools / resources / prompts

```bash
ov eval mcp list-tools jupyter
# list_notebooks       List all notebooks accessible in the workspace.
# execute_cell         Execute a cell and return its outputs.
# insert_cell          Insert a new cell at the given position.
# …

ov eval mcp list-tools jupyter --json | jq '.[].name'
```

Plaintext output is tab-separated (`name\tfirst-line-of-description`) for easy `contains:` matching in declarative tests without JSON parsing. Multi-line descriptions collapse to the first non-empty line. `list-resources` emits `uri\tname\tmime`; `list-prompts` emits `name\tdescription`. The runner automatically pages through `NextCursor` so all results return in a single invocation.

### Call a tool

```bash
ov eval mcp call jupyter list_notebooks '{}'
# [{"path":"finetuning/01_FastInference_Qwen.ipynb",…}]

ov eval mcp call jupyter get_cell '{"path":"intro.ipynb","index":0}'
```

The third positional is the tool's arguments as a JSON object (optional — omit for zero-arg tools). The argument string is parsed with `encoding/json`; parse errors exit 1 immediately with the error location. Returned `TextContent` blocks print one per line; `ImageContent` / `AudioContent` emit a `[image content: <mime>, <N> bytes]` placeholder (use `--json` for the actual payload). If the server sets `IsError: true`, the leaf exits 1 after printing the error text to stderr.

### Read a resource

```bash
ov eval mcp read my-image file:///workspace/data.txt
```

Calls `ReadResource(URI: …)`. Text content is printed to stdout; binary blobs emit a `[binary resource … use --json for base64]` placeholder to stderr.

## Disambiguating multiple servers

When an image declares more than one `mcp_provides` entry, every leaf requires `--name`:

```bash
ov eval mcp ping my-image
# ov: error: image provides multiple mcp servers; use --name (available: jupyter, chrome-devtools)

ov eval mcp ping my-image --name chrome-devtools
# ok
```

In declarative tests, the modifier is `mcp_name:`:

```yaml
- mcp: ping
  mcp_name: chrome-devtools
  scope: deploy
```

No existing image ships multiple providers today — `jupyter` layers expose one server (`jupyter`), `chrome-devtools-mcp` exposes one (`chrome-devtools`). The disambiguation exists for future compositions.

## Port-publishing gotcha (encountered during live testing)

**Symptom:** `ov eval mcp ping` fails with:

```
ov: error: mcp chrome-devtools: container port 9224/tcp is not published to a host port;
declare `ports: [9224:9224]` in the image or run the test from inside the pod
```

**Cause:** the image's OCI label declares the port (e.g., `chrome-devtools-mcp` adds `9224` to `sway-browser-vnc`'s port list), but the running container's quadlet doesn't publish it because `deploy.yml`'s per-image `ports:` entry **replaces** (not appends to) the image-declared list. When a new mcp-providing layer is added to an image that already has a `deploy.yml` entry with an explicit `ports:` list, the new port silently drops out.

**Fix:** either add the mcp port to the instance's `deploy.yml` entry:

```yaml
# ~/.config/ov/deploy.yml
images:
  sway-browser-vnc:
    ports:
      - 5900:5900
      - 9250:9222
      - 9224:9224   # ← add this
```

… or remove the `ports:` override entirely so the image default (which already includes 9224) applies. Re-run `ov config <image>` and restart the service (`ov stop && ov start` — `ov update` may no-op if the image tag hasn't changed).

**Verification** that the fix worked:

```bash
podman inspect ov-<image> --format '{{json .NetworkSettings.Ports}}' | jq
# Expect: the mcp port maps to a non-null [{"HostIp":"...","HostPort":"..."}]
ov eval mcp ping <image>
# ok
```

Instance-specific deployments (the `-i <instance>` form) typically have their ports declared at create-time and are less prone to this drift.

## Transport dispatch

| Declared `transport:` | SDK transport | Notes |
|-----------------------|---------------|-------|
| `http` or empty | `StreamableClientTransport{Endpoint}` | Project default; jupyter-mcp and chrome-devtools-mcp both speak this |
| `streamable`, `streamable-http` | `StreamableClientTransport{Endpoint}` | Aliases |
| `sse` | `SSEClientTransport{Endpoint}` | Legacy SSE servers |
| anything else | error | `"unsupported mcp transport %q (expected http or sse)"` |

The SDK's transports are struct-literal constructors — there is no `NewStreamableClientTransport(...)` helper. Setting `Endpoint` is the only required field.

## Output format: plaintext by default

Every leaf's default format is author-friendly plaintext with one record per line:

- `ping` → `ok`
- `list-tools` → `<name>\t<first-line-of-description>`
- `list-resources` → `<uri>\t<name>\t<mime-type>`
- `list-prompts` → `<name>\t<description>`
- `call` → concatenated `TextContent.Text` payloads, one per line
- `read` → concatenated `ResourceContents.Text`, one per line
- `servers` → `<name>\t<url>\t<transport>`

`--json` on any leaf emits the SDK's raw result struct (e.g., `ListToolsResult`, `CallToolResult`) with `json.MarshalIndent`. Declarative tests always receive plaintext — the subprocess dispatcher runs `ov eval mcp …` without `--json`.

## Declarative authoring examples

Tests currently shipping in the three provider layers (`layers/jupyter/layer.yml`, `layers/jupyter-ml/layer.yml`, `layers/chrome-devtools-mcp/layer.yml`):

```yaml
# Liveness check — fastest sanity verification
- id: mcp-jupyter-ping
  scope: deploy
  mcp: ping
  timeout: 10s

# Catalog assertion — ensure the server exposes the tools we expect
- id: mcp-jupyter-list-tools
  scope: deploy
  mcp: list-tools
  stdout:
    - contains: insert_cell
    - contains: execute_cell

# Real tool invocation — exercises the full request/response path
- id: mcp-jupyter-list-notebooks
  scope: deploy
  mcp: call
  tool: list_notebooks
  input: "{}"
  exit_status: 0

# Tool with arguments
- id: mcp-get-notebook
  scope: deploy
  mcp: call
  tool: get_notebook
  input: '{"path":"getting-started.ipynb"}'
  stdout:
    - contains: cells

# Resource read
- id: mcp-read-prompt-template
  scope: deploy
  mcp: read
  uri: file:///workspace/prompt.txt
  stdout:
    - matches: "."
```

**Deploy-scope only.** `mcp:` checks require a running container with the mcp port published; `ov image validate` rejects build-scope mcp checks at authoring time, and `ov eval image` skips them at runtime with the message `"mcp: <method> requires a running container (skip under ov eval image)"`. Follow the same rule as the other four live-container verbs — `cdp`, `wl`, `dbus`, `vnc`.

## Validator coverage

`ov image validate` enforces at the `validateOvVerb` dispatch in `ov/validate_tests.go`:

- Method name must be in `mcpMethods` (7 entries); unknown methods list the allowed set in the error.
- `scope` must be `deploy`; build-scope raises `"mcp: verb requires scope:\"deploy\""`.
- `call` requires `tool:`; missing field raises `"mcp: call requires modifier \"tool\""`.
- `read` requires `uri:`.

No other required modifiers — `ping`, `servers`, `list-*` take only the optional `mcp_name:`.

---

# Part 2 — Server (`ov mcp serve`)

## Overview

`ov mcp serve` runs the ov CLI *as* an MCP server. Every leaf command in the Kong CLI tree — `image.build`, `status`, `test.mcp.ping`, `config.setup`, `image.new.project`, `layer.add-rpm`, etc. — becomes a callable MCP tool. Tool catalogs are **auto-generated from Kong struct tags** by reflection; there is no hand-written schema per command. Result: **190 tools** covering the entire build + test + deploy surface, including the MCP-first authoring verbs (14 added in 2026: `image.{new.project, new.image, set, add-layer, rm-layer, fetch, refresh, write, cat}` + `layer.{set, add-rpm, add-deb, add-pac, add-aur}`).

```bash
ov mcp serve                                # Streamable HTTP on :18765/mcp
ov mcp serve --listen 127.0.0.1:9999        # Custom port
ov mcp serve --path /api/mcp                # Custom HTTP path prefix (default /mcp)
ov mcp serve --stdio                        # Stdio transport for editor/LLM integration
ov mcp serve --read-only                    # Skip registering the 51 destructive tools
```

The server uses the same `github.com/modelcontextprotocol/go-sdk` v1.5.0 as the client, so the wire format is identical and `ov eval mcp ping <image>` works against it unchanged.

## Architecture

The server lives in a single file: `ov/mcp_server.go`.

1. **Reflection** — `buildMcpServer(readOnly)` calls `kong.New(&CLI{}, …)` to materialise the CLI tree, then walks `k.Model.Leaves(true)` — every non-branch node. For each leaf, `kongLeafToTool(leaf, path, destructive)` emits an `*mcp.Tool`:
   - **Name**: dot-joined Kong path (e.g. `image.build`, `test.mcp.ping`, `config.setup`).
   - **Description**: Kong's `Help` tag, with a `[destructive: …]` annotation appended for mutating tools.
   - **InputSchema**: JSON schema built from `long:""`, `help:""`, `enum:""`, `default:""`, `required:""` struct tags. Positional args become required properties; flags become optional properties. **Every schema has `additionalProperties: false`** — unknown keys are rejected by the SDK's input validation before the handler runs. The schema validator is LLM-honest about the allowed surface.
   - **Annotations**: destructive tools get `DestructiveHint: &true`; everything else gets `ReadOnlyHint: true`.

2. **Destructive gating** — `mcpDestructivePaths` is an explicit 63-entry allowlist of mutating tools (lifecycle: `remove`/`stop`/`start`/`update`/`cmd`/`shell`/`service.*`; config: `config.setup`/`mount`/`unmount`/`passwd`/`remove`; secrets: `set`/`delete`/`import`/`init`/`gpg.setup`/`gpg.set`/`gpg.unset`/`gpg.edit`/`gpg.encrypt`/`gpg.add-recipient`/`gpg.import-key`; deploy: `import`/`reset`; image build/scaffold/edit: `image.build`/`merge`/`new.{layer,project,image}`/`set`/`add-layer`/`rm-layer`/`refresh`/`write`; layer edit: `layer.set`/`add-rpm`/`add-deb`/`add-pac`/`add-aur`; VM: `create`/`destroy`/`start`/`stop`/`build`; udev: `install`/`remove`; alias: `install`/`uninstall`/`add`/`remove`; record: `start`/`stop`/`cmd`; tmux: `kill`/`run`/`send`/`cmd`; settings: `set`/`reset`/`migrate-secrets`). When `--read-only` is set, `buildMcpServer` skips registration entirely rather than gating at runtime — read-only servers expose **127 tools** (190 − 63). `image.fetch` (idempotent, additive cache prime) and `image.cat` (read-only file read) are **not** in the destructive set despite living in the authoring family — they're safe under `--read-only`.

3. **Tool invocation** — `makeToolHandler(path, leaf)` returns a closure. On call: decode the MCP JSON arguments, reconstruct a `[]string` argv via `argvFromJSON(…)` (booleans → bare flag, slices → repeated `--flag=value`, positionals in Kong order), then `captureAndRun(argv)` builds a fresh `kong.New(&CLI{})`, calls `k.Parse(argv)`, invokes `kctx.Run()`, and returns captured stdout/stderr as a single `TextContent`. Errors become `IsError: true` tool results, not MCP-protocol errors — the LLM sees the failure text.

4. **Capture model — os.Stdout / os.Stderr redirect, NOT fd-level** — the first implementation used `syscall.Dup2(pipe, fd=1)` for fd-level capture (would also catch subprocess output and Go's builtin `println`). It **deadlocked** because the MCP SDK's `StdioTransport` captures `os.Stdout` by pointer at Connect time — after dup2'ing fd 1 to our capture pipe, the SDK's JSON-RPC responses went into the tool-output buffer instead of reaching the client. The fix: redirect the `os.Stdout`/`os.Stderr` package variables only. This requires every ov command to write via `fmt.Println`/`fmt.Printf` (not Go's builtin `println`, which bypasses `os.Stderr` and writes directly to fd 2). The `VersionCmd` fix (`println` → `fmt.Println`) landed with this work so `ov version` now routes through the capture pipe correctly. Subprocess output *is* a known leak — but the `ov` codebase invokes subprocesses via `cmd.Stdout = os.Stderr` etc., so in practice it stays captured.

5. **Serialisation** — `runMu` (sync.Mutex) serialises handler invocations because `os.Stdout`/`os.Stderr` redirects are global. MCP tool calls on the server run sequentially; most tools are I/O-bound shell-outs to podman so parallelism is not a big loss.

6. **Transport** — `mcp.NewStreamableHTTPHandler(func(*http.Request) *mcp.Server { return server }, nil)` wraps the server for HTTP mode. In stdio mode, `server.Run(ctx, &mcp.StdioTransport{})` blocks on stdin.

## Deployment: the `ov-mcp` layer

Composing `ov-mcp` into an image deploys the server via supervisord:

```yaml
# layers/ov-mcp/layer.yml (summary)
layers:
  - ov
  - supervisord
ports:
  - 18765
mcp_provides:
  - name: ov
    url: "http://{{.ContainerName}}:18765/mcp"
    transport: http
volumes:
  - name: project        # Bind-mount the project root; see below
    path: /workspace
env:
  OV_PROJECT_DIR: "/workspace"
services:
  - name: ov-mcp
    exec: /usr/local/bin/ov mcp serve --listen :18765
    restart: always
    enable: true
    scope: system
```

**Project-dir wiring** — build-mode tools (`image.build`, `image.inspect`, `image.list.*`) resolve `image.yml` via `os.Getwd()`. Inside the container, cwd is `/workspace` (set by the `ov-mcp` layer's `OV_PROJECT_DIR` env + `volumes:` declaration). Three deployment patterns, in order of progressively less local setup:

1. **Bind-mount** — the canonical `ov-mcp` pattern. The layer ships `env: OV_PROJECT_DIR: /workspace` + `volumes: project → /workspace`; the deployer attaches a host checkout via `ov config <image> --bind project=/path/to/overthink`. The ov CLI's global `-C` / `--dir` / `OV_PROJECT_DIR` flag honours the env var before Kong dispatch, calling `os.Chdir(OV_PROJECT_DIR)` once. Use this when the agent should see your in-flight local edits. The volume NAME stays `project` (stable bind-mount API) — only the in-container PATH changed from `/project` to `/workspace` in 2026-04 (the generic name works whether the contents are an overthink checkout or any other workspace).

2. **Remote pin** — set `OV_PROJECT_REPO=overthinkos/overthink@<sha-or-ref>` in the container env (e.g. via `ov config <image> -e OV_PROJECT_REPO=...`). The ov CLI clones (or hits its `~/.cache/ov/repos/` cache) and chdirs into the cache path before Kong dispatch. No bind mount required. Use this for reproducible agent runs against a pinned upstream.

3. **Auto-default** — `ov mcp serve` with no image.yml reachable at cwd silently falls back to `github.com/overthinkos/overthink`. **Changed in 2026-04:** the fallback now fires regardless of `OV_PROJECT_DIR` being set — it checks whether the resolved cwd actually contains `image.yml`, not whether the env var is populated. This matters because the `ov-mcp` layer permanently sets `OV_PROJECT_DIR=/workspace`; the old early-return-on-env-set logic meant the auto-fallback was effectively dead code for any ov-mcp container. Now a deployer who forgets the `--bind` still gets a working MCP server backed by the upstream repo, with a log line naming the reason (`ov mcp: OV_PROJECT_DIR=/workspace has no image.yml; falling back to default repo …`). Pass `--no-default-repo` on the serve command to opt out (hard-fail with a clear message). This is the only command in the entire CLI that auto-fetches; the top-level CLI stays opt-in. Implementation: `McpServeCmd.bootstrapProject()` in `ov/mcp_server.go`.

See `/ov-build:image` "Project directory resolution" for the flag/env semantics, and `ov/mcp_serve_default_repo_test.go` for the auto-fallback behaviour test.

**Composition style** — `ov-mcp` uses `layers: [ov, supervisord]` (meta-layer composition) rather than `depends:` (hard prerequisite) because it adds no install of its own — it's pure wiring. Images that want the MCP server add `ov-mcp` to their layer list; images that just want the ov binary continue to use the `ov` layer alone. Both `layers:` and `depends:` reference other layers, but only `layers:` lets the using layer ship no install files.

## Verifying end to end

```bash
# Build an ov-mcp-bearing image (e.g. arch-ov) and start it:
ov image build arch-ov
ov config arch-ov --bind project=/home/you/overthink
ov start arch-ov

# From the host — the container's mcp_provides URL is auto-rewritten to the published host port:
ov eval mcp ping arch-ov --name ov
# ok
ov eval mcp list-tools arch-ov --name ov | wc -l
# 190
ov eval mcp call arch-ov version '{}' --name ov
# 2026.nnn.nnnn          (the container's own ov version)
ov eval mcp call arch-ov image.list.images '{}' --name ov
# arch-ov [testing]      (reads image.yml from the bind-mounted /workspace)
# archlinux [testing]
# …
```

The deploy-scope tests in `layers/ov-mcp/layer.yml` cover this exact sequence: service-running, port-reachable, `mcp: ping`, `mcp: list-tools`, `mcp: call tool=version`, `mcp: call tool=image.list.images` (the last proves the bind-mount + OV_PROJECT_DIR wiring).

## Authoring tools (build-from-scratch over MCP)

Every CLI verb under `ov image …` and `ov layer …` auto-becomes an MCP tool via Kong reflection. The authoring surface added for "build a project from scratch using only `ov mcp`" exposes these tools:

| MCP tool | What it does |
|---|---|
| `image.new.project` | Scaffold `image.yml` (referencing the upstream `build.yml` remotely), `layers/`, `.gitignore`. |
| `image.new.image` | Append a new image entry to `image.yml`. |
| `image.new.layer` | Scaffold `layers/<name>/layer.yml` with a stub. |
| `image.set` | Set any value in `image.yml` by dot-path (`defaults.tag`, `images.foo.layers`, …). Value is parsed as YAML. |
| `image.add-layer` / `image.rm-layer` | Append / remove a layer from an image's `layers:` list (idempotent). |
| `layer.set` | Set any value in `layers/<name>/layer.yml` by dot-path. |
| `layer.add-rpm` / `layer.add-deb` / `layer.add-pac` / `layer.add-aur` | Append packages to a layer's `<format>.packages` list. Idempotent. Upgrades scaffold's null `packages:` value to a real sequence. |
| `image.fetch` / `image.refresh` | Pre-prime / re-clone the remote-repo cache. Spec defaults to `default` (overthinkos/overthink). |
| `image.write` / `image.cat` | Write / read any file under the project root — escape hatch for free-form auxiliary files (`pixi.toml`, `package.json`, `root.yml`, scripts, `*.service`). Path is resolved against `os.Getwd()` and rejected if it escapes the project root. |

All YAML edits go through the `yaml.v3` *node* API (not value unmarshal) so comments and key order are preserved across edits. Implementation in `ov/scaffold_project.go`, `ov/yaml_setter.go`, and `ov/scaffold_cmds.go`. Tested in `ov/scaffold_project_test.go` and `ov/yaml_setter_test.go`.

End-to-end MCP-only worked example:

```jsonc
// All called as MCP tool calls (e.g. via `ov eval mcp call <ctr> <tool> '<args>'`):
image.new.project   {"dir": "/tmp/hello"}
image.new.image     {"name": "hello", "base": "quay.io/fedora/fedora:43"}
image.new.layer     {"name": "hello-svc"}
layer.add-rpm       {"name": "hello-svc", "packages": ["openssh-server"]}
image.add-layer     {"image": "hello", "layer": "hello-svc"}
image.validate      {}
image.build         {"image": "hello"}
image.inspect       {"image": "hello"}
```

## Port choice

Default `:18765` chosen for non-collision with other MCP layers:
- `8888` — jupyter-mcp
- `9224` — chrome-devtools-mcp (via mcp-proxy)
- `18789` — openclaw gateway

## Policy note

The server registers destructive tools with `DestructiveHint: true` rather than withholding them. The LLM runtime (Claude Code, Open WebUI) is responsible for acting on the hint — e.g. prompting the user before calling an annotated tool. For hostile-LLM scenarios or untrusted network deployments, run with `--read-only` (drops the 51 mutating tools at registration time) and/or restrict reach via the tunnel / Traefik layer.

---

## Cross-References

- `/ov-build:eval` — parent router; all `ov eval mcp …` invocations are dispatched through it. Full method allowlist for all 5 live-container verbs (cdp/wl/dbus/vnc/mcp) lives in the "Live-container verb catalog" section.
- `/ov-build:layer` — `mcp_provides` / `mcp_accepts` / `mcp_requires` field reference for layer authoring.
- `/ov-core:config` — how `mcp_provides` gets injected into `deploy.yml` `provides.mcp:` and synthesized into `OV_MCP_SERVERS` for consumers at `ov config` time; pod-aware resolution to `localhost`; instance-aware MCP server naming with `-<instance>` suffix.
- `/ov-build:validate` — authoring-time validation rules (method allowlist, required modifiers, scope enforcement).
- `/ov-core:deploy` — `deploy.yml` `ports:` override semantics (the port-publishing gotcha lives here operationally).
- `/ov-advanced:cdp` — sibling live-container verb (Chrome DevTools Protocol).
- `/ov-advanced:wl` — sibling (Wayland desktop control).
- `/ov-advanced:dbus` — sibling (D-Bus calls/notifications).
- `/ov-advanced:vnc` — sibling (VNC framebuffer / input).
- `/ov-jupyter:jupyter-mcp` — the FastMCP server implementation layer (13 tools for notebook manipulation over CRDT).
- `/ov-selkies:chrome-devtools-mcp` — the mcp-proxy wrapper around chrome-devtools-mcp (29 tools for browser automation).
- `/ov-hermes:hermes` — a consumer (`mcp_accepts: jupyter, chrome-devtools`); use `ov eval mcp` to verify the services hermes discovers are actually alive.
- `/ov-openwebui:openwebui` — another consumer (`mcp_accepts: jupyter, chrome-devtools`).
- `/ov-jupyter:jupyter`, `/ov-jupyter:jupyter-ml`, `/ov-jupyter:jupyter-ml-notebook` — images bundling `jupyter-mcp`; `ov test <image> --filter mcp` exercises the verb end-to-end.
- `/ov-selkies:sway-browser-vnc`, `/ov-selkies:selkies-desktop`, `/ov-selkies:selkies-desktop-nvidia` — images bundling `chrome-devtools-mcp` (transitively via the chrome metalayer).
- `/ov-dev:go` — implementation map: `mcp.go` (client Kong subcommand tree), `mcp_client.go` (client SDK wrapper + URL rewriter), `mcp_server.go` (server: Kong→MCP reflection, destructive-hint set, `captureAndRun`), `testrun_ov_verbs.go` (declarative dispatcher entry `mcpMethods`), `validate_tests.go` (`validateOvVerb` case for `mcp`).
- `/ov-coder:ov-mcp` — the deployment layer that wires `ov mcp serve` into an image via supervisord. Includes the `/workspace` bind-mount (volume NAME `project`) + `OV_PROJECT_DIR` env var pattern for build-mode tools.
- `/ov-foundation:ov` — the underlying binary layer; required by `ov-mcp`.
- `/ov-build:image` — "Project directory resolution" subsection documents the `-C` / `--dir` / `OV_PROJECT_DIR` global flag that makes the server's project-dir bind-mount work.
- `/ov-coder:arch-ov`, `/ov-foundation:fedora-ov` — canonical images composing `ov-mcp` for remote ov-via-MCP deployments.

## When to Use This Skill

**MUST be invoked** when the task involves Model Context Protocol on either side:

- **Client** — `ov eval mcp` commands, probing/testing MCP servers declared via `mcp_provides`, examining MCP tool/resource/prompt catalogs, debugging the URL-rewriter or port-publishing behavior, or authoring deploy-scope `mcp:` checks in a `tests:` block.
- **Server** — `ov mcp serve` operation, the `ov-mcp` layer, destructive-hint policy, the `--read-only` filter, Kong-reflection tool generation, the project-dir bind-mount pattern, or symptoms like "MCP tool returned empty output" (check `println` vs `fmt.Println` in the invoked command — the server captures `os.Stdout`, not fd 1).

Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Test mode / live service operation. Client: deploy an image with `mcp_provides`, start it, then probe. Server: compose `ov-mcp` into an image, bind `--bind project=/path/to/overthink` at config time, start the container, then consume from any MCP-speaking LLM runtime. Pairs with `/ov-build:eval` (parent router + declarative-verb catalog), `/ov-core:config` (consumer-side injection + `--bind` for the project dir), `/ov-coder:ov-mcp` (server deployment layer), and the consumer-layer skills (`/ov-hermes:hermes`, `/ov-openwebui:openwebui`).
