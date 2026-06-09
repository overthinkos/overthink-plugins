---
name: charly-mcp-cmd
description: |
  MUST be invoked before any work involving: Model Context Protocol — both directions. (1) `charly eval mcp` client: probing MCP servers declared via mcp_provide, testing MCP tool catalogs, debugging the URL-rewriter (including host-networked containers via `HostConfig.NetworkMode` detection) or port-publishing behavior. (2) `charly mcp serve` server: running the charly CLI itself as an MCP server over Streamable HTTP or stdio, auto-generated from Kong reflection (~192 tools including the MCP-first authoring surface — image/layer scaffolding, comment-preserving YAML edits, free-form file writes), destructive-hint annotations, the `--read-only` filter, auto-fallback to `overthinkos/overthink` when cwd has no `box.yml` (always fires regardless of CHARLY_PROJECT_DIR being set), and the `charly-mcp` deployment layer with its `/workspace` bind mount.
  Named `charly-mcp-cmd` (not `mcp`) to disambiguate from Claude Code's built-in `/mcp` slash command (the `-cmd` suffix avoids collision with the existing `/charly-coder:charly-mcp` image skill).
---

# MCP - Model Context Protocol (client + server)

`charly` speaks MCP in both directions. This skill covers both:

- **Client** — `charly eval mcp <method>`: connect to any MCP server declared via `mcp_provide` and probe/call/read. Used to test MCP endpoints shipped by `jupyter-mcp`, `chrome-devtools-mcp`, or the charly server itself.
- **Server** — `charly mcp serve`: expose the entire charly CLI surface (build + test + deploy modes, 190 tools) as MCP over Streamable HTTP or stdio. Used by LLM agents (Claude Code, Open WebUI, OpenClaw) to drive charly remotely. Deployed in-container via the `charly-mcp` layer. The 190-tool catalog now includes a project-scaffolding + YAML-editing + file-write authoring surface, so an agent can build an `charly` project from scratch over RPC — see "Authoring tools" below.

Both surfaces share the same SDK: `github.com/modelcontextprotocol/go-sdk v1.5.0`.

---

# Part 1 — Client (`charly eval mcp …`)

## Overview

`charly eval mcp` connects to Model Context Protocol servers declared by running containers via `mcp_provide`, using [github.com/modelcontextprotocol/go-sdk](https://pkg.go.dev/github.com/modelcontextprotocol/go-sdk) (v1.5.0). Seven leaf subcommands cover the full MCP client surface: `ping`, `servers`, `list-tools`, `list-resources`, `list-prompts`, `call`, `read`. No MCP URL argument is ever typed by the user — the verb reads the target image's `ai.opencharly.mcp_provide` OCI label, resolves `{{.ContainerName}}` templates, applies pod-aware `localhost` rewriting, and maps the container-network URL to the published host port automatically.

### Also as a declarative verb

Every `charly eval mcp <method>` is authorable as an `mcp:` verb inside a `eval:` block. The method name becomes the verb's YAML value; method-specific args are sibling fields (`tool:`, `uri:`, `input:`, `mcp_name:`). Shared matchers (`stdout:`, `stderr:`, `exit_status:`, `timeout:`) work like other verbs. See `/charly-eval:eval` for the parent router and the complete method allowlist. Example:

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
| Ping | `charly eval mcp ping <image>` | Liveness check — returns `ok` on a successful `Ping` RPC |
| Enumerate | `charly eval mcp servers <image>` | List MCP servers declared by the image (no dial) |
| List tools | `charly eval mcp list-tools <image>` | Tool name + first-line description per line |
| List resources | `charly eval mcp list-resources <image>` | URI + name + MIME type per line |
| List prompts | `charly eval mcp list-prompts <image>` | Prompt name + description per line |
| Call | `charly eval mcp call <image> <tool> [<input-json>]` | Invoke a tool; prints TextContent payload |
| Read | `charly eval mcp read <image> <uri>` | Read a resource; prints Text content |

Every leaf accepts:
- `-i <instance>` — target a specific instance
- `--name <server>` — disambiguate when the image declares multiple `mcp_provide` entries
- `--json` — emit the SDK's native result struct instead of plaintext
- `--timeout <duration>` — per-operation timeout (default 30s)

## Architecture

The verb runs entirely on the host — no delegation into the container — and builds a single `*mcp.ClientSession` per invocation.

1. **Container resolution**: `resolveContainer(image, instance)` → `charly-<image>[-<instance>]`.
2. **Image ref + metadata**: `containerImageRef` → `ExtractMetadata` → `meta.MCPProvide` (read from OCI label `ai.opencharly.mcp_provide`).
3. **Template substitution**: any `{{.ContainerName}}` in the URL is replaced with the resolved container name.
4. **Pod-aware rewrite**: `podAwareMCPProvides` folds same-image entries so the URL host becomes `localhost` (identical to the CHARLY_MCP_SERVERS path used by `charly config`).
5. **Host-side port rewrite**: the load-bearing piece. `rewriteMCPURLForHost` parses the URL; if the host is the container name or `localhost`, it looks up the published host port for the URL's port via `podman inspect` (same `NetworkSettings.Ports` data that powers `${HOST_PORT:N}` in declarative tests) and rewrites to `127.0.0.1:<host-port>`. Non-matching hosts (external URLs) pass through unchanged.
6. **Transport pick**: `transport: http` (or empty) → `StreamableClientTransport{Endpoint}`; `transport: sse` → `SSEClientTransport{Endpoint}`; any other string errors at dial time with the declared transport echoed.
7. **Session**: `mcp.NewClient(&mcp.Implementation{Name: "charly", Version: ComputeCalVer()}, nil).Connect(ctx, transport, nil)` — runs the MCP `initialize` handshake and returns an opened `ClientSession`. A `defer session.Close()` always fires.

Source: `charly/mcp.go` (Kong subcommand tree + leaf Run() methods), `charly/mcp_client.go` (resolver, URL rewriter, transport picker, formatters). The URL rewriter is the one truly novel piece; the rest composes existing infrastructure. See `/charly-internals:go` "Declarative Testing" section for the implementation map.

## Commands

### Ping

```bash
charly eval mcp ping jupyter
# ok
```

Calls `ClientSession.Ping(ctx, nil)`. Exit 0 iff the server responds. Useful as a liveness check, often paired with a short `--timeout` in CI.

### Servers (discovery only, no dial)

```bash
charly eval mcp servers jupyter
# jupyter   http://localhost:8888/mcp   http

charly eval mcp servers jupyter --json
# [{"name":"jupyter","url":"http://localhost:8888/mcp","transport":"http","source":"jupyter"}]
```

Reads `mcp_provide` from the OCI label, applies template substitution + pod-aware rewrite, prints the result. No MCP handshake. Useful for confirming which server names the image advertises before dialing.

### List tools / resources / prompts

```bash
charly eval mcp list-tools jupyter
# list_notebooks       List all notebooks accessible in the workspace.
# execute_cell         Execute a cell and return its outputs.
# insert_cell          Insert a new cell at the given position.
# …

charly eval mcp list-tools jupyter --json | jq '.[].name'
```

Plaintext output is tab-separated (`name\tfirst-line-of-description`) for easy `contains:` matching in declarative tests without JSON parsing. Multi-line descriptions collapse to the first non-empty line. `list-resources` emits `uri\tname\tmime`; `list-prompts` emits `name\tdescription`. The runner automatically pages through `NextCursor` so all results return in a single invocation.

### Call a tool

```bash
charly eval mcp call jupyter list_notebooks '{}'
# [{"path":"finetuning/01_FastInference_Qwen.ipynb",…}]

charly eval mcp call jupyter get_cell '{"path":"intro.ipynb","index":0}'
```

The third positional is the tool's arguments as a JSON object (optional — omit for zero-arg tools). The argument string is parsed with `encoding/json`; parse errors exit 1 immediately with the error location. Returned `TextContent` blocks print one per line; `ImageContent` / `AudioContent` emit a `[image content: <mime>, <N> bytes]` placeholder (use `--json` for the actual payload). If the server sets `IsError: true`, the leaf exits 1 after printing the error text to stderr.

### Read a resource

```bash
charly eval mcp read my-image file:///workspace/data.txt
```

Calls `ReadResource(URI: …)`. Text content is printed to stdout; binary blobs emit a `[binary resource … use --json for base64]` placeholder to stderr.

## Disambiguating multiple servers

When an image declares more than one `mcp_provide` entry, every leaf requires `--name`:

```bash
charly eval mcp ping my-image
# charly: error: image provides multiple mcp servers; use --name (available: jupyter, chrome-devtools)

charly eval mcp ping my-image --name chrome-devtools
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

**Symptom:** `charly eval mcp ping` fails with:

```
charly: error: mcp chrome-devtools: container port 9224/tcp is not published to a host port;
declare `ports: [9224:9224]` in the image or run the test from inside the pod
```

**Cause:** the image's OCI label declares the port (e.g., `chrome-devtools-mcp` adds `9224` to `sway-browser-vnc`'s port list), but the running container's quadlet doesn't publish it because `deploy.yml`'s per-image `port:` entry **replaces** (not appends to) the image-declared list. When a new mcp-providing layer is added to an image that already has a `deploy.yml` entry with an explicit `port:` list, the new port silently drops out.

**Fix:** either add the mcp port to the instance's `deploy.yml` entry:

```yaml
# ~/.config/charly/deploy.yml
images:
  sway-browser-vnc:
    ports:
      - 5900:5900
      - 9250:9222
      - 9224:9224   # ← add this
```

… or remove the `port:` override entirely so the image default (which already includes 9224) applies. Re-run `charly config <image>` and restart the service (`charly stop && charly start` — `charly update` may no-op if the image tag hasn't changed).

**Verification** that the fix worked:

```bash
podman inspect charly-<image> --format '{{json .NetworkSettings.Ports}}' | jq
# Expect: the mcp port maps to a non-null [{"HostIp":"...","HostPort":"..."}]
charly eval mcp ping <image>
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

`--json` on any leaf emits the SDK's raw result struct (e.g., `ListToolsResult`, `CallToolResult`) with `json.MarshalIndent`. Declarative tests always receive plaintext — the subprocess dispatcher runs `charly eval mcp …` without `--json`.

## Declarative authoring examples

Tests currently shipping in the three provider layers (`candy/jupyter/candy.yml`, `candy/jupyter-ml/candy.yml`, `candy/chrome-devtools-mcp/candy.yml`):

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

**Deploy-scope only.** `mcp:` checks require a running container with the mcp port published; `charly box validate` rejects build-scope mcp checks at authoring time, and `charly eval box` skips them at runtime with the message `"mcp: <method> requires a running container (skip under charly eval box)"`. Follow the same rule as the other four live-container verbs — `cdp`, `wl`, `dbus`, `vnc`.

## Validator coverage

`charly box validate` enforces at the `validateCharlyVerb` dispatch in `charly/validate_tests.go`:

- Method name must be in `mcpMethods` (7 entries); unknown methods list the allowed set in the error.
- `scope` must be `deploy`; build-scope raises `"mcp: verb requires scope:\"deploy\""`.
- `call` requires `tool:`; missing field raises `"mcp: call requires modifier \"tool\""`.
- `read` requires `uri:`.

No other required modifiers — `ping`, `servers`, `list-*` take only the optional `mcp_name:`.

---

# Part 2 — Server (`charly mcp serve`)

## Overview

`charly mcp serve` runs the charly CLI *as* an MCP server. Every leaf command in the Kong CLI tree — `image.build`, `status`, `test.mcp.ping`, `config.setup`, `box.new.project`, `candy.add-rpm`, etc. — becomes a callable MCP tool. Tool catalogs are **auto-generated from Kong struct tags** by reflection; there is no hand-written schema per command. Result: **190 tools** covering the entire build + test + deploy surface, including the MCP-first authoring verbs (`image.{new.project, new.image, set, add-layer, rm-layer, fetch, refresh, write, cat}` + `layer.{set, add-rpm, add-deb, add-pac, add-aur}`).

```bash
charly mcp serve                                # Streamable HTTP on :18765/mcp
charly mcp serve --listen 127.0.0.1:9999        # Custom port
charly mcp serve --path /api/mcp                # Custom HTTP path prefix (default /mcp)
charly mcp serve --stdio                        # Stdio transport for editor/LLM integration
charly mcp serve --read-only                    # Skip registering the 51 destructive tools
```

The server uses the same `github.com/modelcontextprotocol/go-sdk` v1.5.0 as the client, so the wire format is identical and `charly eval mcp ping <image>` works against it unchanged.

## Architecture

The server lives in a single file: `charly/mcp_server.go`.

1. **Reflection** — `buildMcpServer(readOnly)` calls `kong.New(&CLI{}, …)` to materialise the CLI tree, then walks `k.Model.Leaves(true)` — every non-branch node. For each leaf, `kongLeafToTool(leaf, path, destructive)` emits an `*mcp.Tool`:
   - **Name**: dot-joined Kong path (e.g. `image.build`, `test.mcp.ping`, `config.setup`).
   - **Description**: Kong's `Help` tag, with a `[destructive: …]` annotation appended for mutating tools.
   - **InputSchema**: JSON schema built from `long:""`, `help:""`, `enum:""`, `default:""`, `required:""` struct tags. Positional args become required properties; flags become optional properties. **Every schema has `additionalProperties: false`** — unknown keys are rejected by the SDK's input validation before the handler runs. The schema validator is LLM-honest about the allowed surface.
   - **Annotations**: destructive tools get `DestructiveHint: &true`; everything else gets `ReadOnlyHint: true`.

2. **Destructive gating** — `mcpDestructivePaths` is an explicit 63-entry allowlist of mutating tools (lifecycle: `remove`/`stop`/`start`/`update`/`cmd`/`shell`/`service.*`; config: `config.setup`/`mount`/`unmount`/`passwd`/`remove`; secrets: `set`/`delete`/`import`/`init`/`gpg.setup`/`gpg.set`/`gpg.unset`/`gpg.edit`/`gpg.encrypt`/`gpg.add-recipient`/`gpg.import-key`; deploy: `import`/`reset`; image build/scaffold/edit: `image.build`/`merge`/`new.{layer,project,image}`/`set`/`add-layer`/`rm-layer`/`refresh`/`write`; layer edit: `candy.set`/`add-rpm`/`add-deb`/`add-pac`/`add-aur`; VM: `create`/`destroy`/`start`/`stop`/`build`; udev: `install`/`remove`; alias: `install`/`uninstall`/`add`/`remove`; record: `start`/`stop`/`cmd`; tmux: `kill`/`run`/`send`/`cmd`; settings: `set`/`reset`/`migrate-secrets`). When `--read-only` is set, `buildMcpServer` skips registration entirely rather than gating at runtime — read-only servers expose **127 tools** (190 − 63). `image.fetch` (idempotent, additive cache prime) and `box.cat` (read-only file read) are **not** in the destructive set despite living in the authoring family — they're safe under `--read-only`.

3. **Tool invocation** — `makeToolHandler(path, leaf)` returns a closure. On call: decode the MCP JSON arguments, reconstruct a `[]string` argv via `argvFromJSON(…)` (booleans → bare flag, slices → repeated `--flag=value`, positionals in Kong order), then `captureAndRun(argv)` builds a fresh `kong.New(&CLI{})`, calls `k.Parse(argv)`, invokes `kctx.Run()`, and returns captured stdout/stderr as a single `TextContent`. Errors become `IsError: true` tool results, not MCP-protocol errors — the LLM sees the failure text.

4. **Capture model — os.Stdout / os.Stderr redirect, NOT fd-level** — capture redirects the `os.Stdout`/`os.Stderr` package variables only, not fd 1/2 via `syscall.Dup2`. Fd-level capture deadlocks: the MCP SDK's `StdioTransport` captures `os.Stdout` by pointer at Connect time, so dup2'ing fd 1 to a capture pipe sends the SDK's JSON-RPC responses into the tool-output buffer instead of to the client. Because the redirect is at the package-variable level, every charly command must write via `fmt.Println`/`fmt.Printf` (not Go's builtin `println`, which bypasses `os.Stderr` and writes directly to fd 2) — `VersionCmd` uses `fmt.Println` for exactly this reason. Subprocess output *is* a known leak — but the `charly` codebase invokes subprocesses via `cmd.Stdout = os.Stderr` etc., so in practice it stays captured.

5. **Serialisation** — `runMu` (sync.Mutex) serialises handler invocations because `os.Stdout`/`os.Stderr` redirects are global. MCP tool calls on the server run sequentially; most tools are I/O-bound shell-outs to podman so parallelism is not a big loss.

6. **Transport** — `mcp.NewStreamableHTTPHandler(func(*http.Request) *mcp.Server { return server }, nil)` wraps the server for HTTP mode. In stdio mode, `server.Run(ctx, &mcp.StdioTransport{})` blocks on stdin.

## Deployment: the `charly-mcp` layer

Composing `charly-mcp` into an image deploys the server via supervisord:

```yaml
# candy/charly-mcp/candy.yml (summary)
candy:
  - charly
  - supervisord
ports:
  - 18765
mcp_provide:
  - name: charly
    url: "http://{{.ContainerName}}:18765/mcp"
    transport: http
volumes:
  - name: project        # Bind-mount the project root; see below
    path: /workspace
env:
  CHARLY_PROJECT_DIR: "/workspace"
service:
  - name: charly-mcp
    exec: /usr/local/bin/charly mcp serve --listen :18765
    restart: always
    enable: true
    scope: system
```

**Project-dir wiring** — build-mode tools (`image.build`, `image.inspect`, `image.list.*`) resolve `box.yml` via `os.Getwd()`. Inside the container, cwd is `/workspace` (set by the `charly-mcp` layer's `CHARLY_PROJECT_DIR` env + `volume:` declaration). Three deployment patterns, in order of progressively less local setup:

1. **Bind-mount** — the canonical `charly-mcp` pattern. The layer ships `env: CHARLY_PROJECT_DIR: /workspace` + `volumes: project → /workspace`; the deployer attaches a host checkout via `charly config <image> --bind project=/path/to/opencharly`. The charly CLI's global `-C` / `--dir` / `CHARLY_PROJECT_DIR` flag honours the env var before Kong dispatch, calling `os.Chdir(CHARLY_PROJECT_DIR)` once. Use this when the agent should see your in-flight local edits. The volume NAME is `project` (stable bind-mount API); the in-container PATH is `/workspace` (the generic name works whether the contents are an opencharly checkout or any other workspace).

2. **Remote pin** — set `CHARLY_PROJECT_REPO=overthinkos/overthink@<sha-or-ref>` in the container env (e.g. via `charly config <image> -e CHARLY_PROJECT_REPO=...`). The charly CLI clones (or hits its `~/.cache/charly/repos/` cache) and chdirs into the cache path before Kong dispatch. No bind mount required. Use this for reproducible agent runs against a pinned upstream.

3. **Auto-default** — `charly mcp serve` with no box.yml reachable at cwd silently falls back to `github.com/overthinkos/overthink`. The fallback fires regardless of `CHARLY_PROJECT_DIR` being set — it checks whether the resolved cwd actually contains `box.yml`, not whether the env var is populated. This matters because the `charly-mcp` layer permanently sets `CHARLY_PROJECT_DIR=/workspace`: a deployer who forgets the `--bind` still gets a working MCP server backed by the upstream repo, with a log line naming the reason (`charly mcp: CHARLY_PROJECT_DIR=/workspace has no box.yml; falling back to default repo …`). Pass `--no-default-repo` on the serve command to opt out (hard-fail with a clear message). This is the only command in the entire CLI that auto-fetches; the top-level CLI stays opt-in. Implementation: `McpServeCmd.bootstrapProject()` in `charly/mcp_server.go`.

See `/charly-image:image` "Project directory resolution" for the flag/env semantics, and `charly/mcp_serve_default_repo_test.go` for the auto-fallback behaviour test.

**Composition style** — `charly-mcp` uses `layers: [charly, supervisord]` (meta-layer composition) rather than `require:` (hard prerequisite) because it adds no install of its own — it's pure wiring. Images that want the MCP server add `charly-mcp` to their layer list; images that just want the charly binary continue to use the `charly` layer alone. Both `layer:` and `require:` reference other layers, but only `layer:` lets the using layer ship no install files.

## Verifying end to end

```bash
# Build an charly-mcp-bearing image (e.g. charly-arch) and start it:
charly box build charly-arch
charly config charly-arch --bind project=/home/you/opencharly
charly start charly-arch

# From the host — the container's mcp_provide URL is auto-rewritten to the published host port:
charly eval mcp ping charly-arch --name charly
# ok
charly eval mcp list-tools charly-arch --name charly | wc -l
# 190
charly eval mcp call charly-arch version '{}' --name charly
# 2026.nnn.nnnn          (the container's own charly version)
charly eval mcp call charly-arch box.list.boxes '{}' --name charly
# charly-arch [testing]      (reads box.yml from the bind-mounted /workspace)
# arch [testing]
# …
```

The deploy-scope tests in `candy/charly-mcp/candy.yml` cover this exact sequence: service-running, port-reachable, `mcp: ping`, `mcp: list-tools`, `mcp: call tool=version`, `mcp: call tool=box.list.boxes` (the last proves the bind-mount + CHARLY_PROJECT_DIR wiring).

## Authoring tools (build-from-scratch over MCP)

Every CLI verb under `charly box …` and `charly candy …` auto-becomes an MCP tool via Kong reflection. The authoring surface added for "build a project from scratch using only `charly mcp`" exposes these tools:

| MCP tool | What it does |
|---|---|
| `box.new.project` | Scaffold `box.yml` (referencing the upstream `build.yml` remotely), `candy/`, `.gitignore`. |
| `box.new.box` | Append a new image entry to `box.yml`. |
| `box.new.candy` | Scaffold `candy/<name>/candy.yml` with a stub. |
| `box.set` | Set any value in `box.yml` by dot-path (`defaults.tag`, `images.foo.layers`, …). Value is parsed as YAML. |
| `box.add-candy` / `box.rm-candy` | Append / remove a layer from an image's `layer:` list (idempotent). |
| `candy.set` | Set any value in `candy/<name>/candy.yml` by dot-path. |
| `candy.add-rpm` / `candy.add-deb` / `candy.add-pac` / `candy.add-aur` | Append packages to a layer's `<format>.packages` list. Idempotent. Upgrades scaffold's null `package:` value to a real sequence. |
| `image.fetch` / `image.refresh` | Pre-prime / re-clone the remote-repo cache. Spec defaults to `default` (overthinkos/overthink). |
| `box.write` / `box.cat` | Write / read any file under the project root — escape hatch for free-form auxiliary files (`pixi.toml`, `package.json`, `root.yml`, scripts, `*.service`). Path is resolved against `os.Getwd()` and rejected if it escapes the project root. |

All YAML edits go through the `yaml.v3` *node* API (not value unmarshal) so comments and key order are preserved across edits. Implementation in `charly/scaffold_project.go`, `charly/yaml_setter.go`, and `charly/scaffold_cmds.go`. Tested in `charly/scaffold_project_test.go` and `charly/yaml_setter_test.go`.

End-to-end MCP-only worked example:

```jsonc
// All called as MCP tool calls (e.g. via `charly eval mcp call <ctr> <tool> '<args>'`):
box.new.project   {"dir": "/tmp/hello"}
box.new.box     {"name": "hello", "base": "quay.io/fedora/fedora:43"}
box.new.candy     {"name": "hello-svc"}
candy.add-rpm       {"name": "hello-svc", "packages": ["openssh-server"]}
box.add-candy     {"image": "hello", "layer": "hello-svc"}
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

- `/charly-eval:eval` — parent router; all `charly eval mcp …` invocations are dispatched through it. Full method allowlist for all 5 live-container verbs (cdp/wl/dbus/vnc/mcp) lives in the "Live-container verb catalog" section.
- `/charly-image:layer` — `mcp_provide` / `mcp_accept` / `mcp_require` field reference for layer authoring.
- `/charly-core:charly-config` — how `mcp_provide` gets injected into `deploy.yml` `provides.mcp:` and synthesized into `CHARLY_MCP_SERVERS` for consumers at `charly config` time; pod-aware resolution to `localhost`; instance-aware MCP server naming with `-<instance>` suffix.
- `/charly-build:validate` — authoring-time validation rules (method allowlist, required modifiers, scope enforcement).
- `/charly-core:deploy` — `deploy.yml` `port:` override semantics (the port-publishing gotcha lives here operationally).
- `/charly-eval:cdp` — sibling live-container verb (Chrome DevTools Protocol).
- `/charly-eval:wl` — sibling (Wayland desktop control).
- `/charly-eval:dbus` — sibling (D-Bus calls/notifications).
- `/charly-eval:vnc` — sibling (VNC framebuffer / input).
- `/charly-jupyter:jupyter-mcp` — the FastMCP server implementation layer (11 tools for notebook manipulation over CRDT: notebook_*/cell_* + notebook_list_users + room_list; clients do not manage CRDT rooms — the server auto-attaches).
- `/charly-selkies:chrome-devtools-mcp` — the mcp-proxy wrapper around chrome-devtools-mcp (29 tools for browser automation).
- `/charly-hermes:hermes` — a consumer (`mcp_accept: jupyter, chrome-devtools`); use `charly eval mcp` to verify the services hermes discovers are actually alive.
- `/charly-openwebui:openwebui` — another consumer (`mcp_accept: jupyter, chrome-devtools`).
- `/charly-jupyter:jupyter`, `/charly-jupyter:jupyter-ml`, `/charly-jupyter:jupyter-ml-notebook` — images bundling `jupyter-mcp`; `charly eval live <image> --filter mcp` exercises the verb end-to-end.
- `/charly-selkies:sway-browser-vnc`, `/charly-selkies:selkies-labwc`, `/charly-selkies:selkies-labwc-nvidia` — images bundling `chrome-devtools-mcp` (transitively via the chrome metalayer).
- `/charly-internals:go` — implementation map: `mcp.go` (client Kong subcommand tree), `mcp_client.go` (client SDK wrapper + URL rewriter), `mcp_server.go` (server: Kong→MCP reflection, destructive-hint set, `captureAndRun`), `testrun_ov_verbs.go` (declarative dispatcher entry `mcpMethods`), `validate_tests.go` (`validateCharlyVerb` case for `mcp`).
- `/charly-coder:charly-mcp` — the deployment layer that wires `charly mcp serve` into an image via supervisord. Includes the `/workspace` bind-mount (volume NAME `project`) + `CHARLY_PROJECT_DIR` env var pattern for build-mode tools.
- `/charly-tools:charly` — the underlying binary layer; required by `charly-mcp`.
- `/charly-image:image` — "Project directory resolution" subsection documents the `-C` / `--dir` / `CHARLY_PROJECT_DIR` global flag that makes the server's project-dir bind-mount work.
- `/charly-coder:charly-arch`, `/charly-distros:charly-fedora` — canonical images composing `charly-mcp` for remote charly-via-MCP deployments.

## When to Use This Skill

**MUST be invoked** when the task involves Model Context Protocol on either side:

- **Client** — `charly eval mcp` commands, probing/testing MCP servers declared via `mcp_provide`, examining MCP tool/resource/prompt catalogs, debugging the URL-rewriter or port-publishing behavior, or authoring deploy-scope `mcp:` checks in a `eval:` block.
- **Server** — `charly mcp serve` operation, the `charly-mcp` layer, destructive-hint policy, the `--read-only` filter, Kong-reflection tool generation, the project-dir bind-mount pattern, or symptoms like "MCP tool returned empty output" (check `println` vs `fmt.Println` in the invoked command — the server captures `os.Stdout`, not fd 1).

Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Test mode / live service operation. Client: deploy an image with `mcp_provide`, start it, then probe. Server: compose `charly-mcp` into an image, bind `--bind project=/path/to/opencharly` at config time, start the container, then consume from any MCP-speaking LLM runtime. Pairs with `/charly-eval:eval` (parent router + declarative-verb catalog), `/charly-core:charly-config` (consumer-side injection + `--bind` for the project dir), `/charly-coder:charly-mcp` (server deployment layer), and the consumer-layer skills (`/charly-hermes:hermes`, `/charly-openwebui:openwebui`).
