---
name: mcp
description: |
  MUST be invoked before any work involving: Model Context Protocol clients, ov test mcp commands, probing MCP servers declared via mcp_provides, testing MCP tool catalogs, or debugging the URL-rewriter / port-publishing behavior for MCP endpoints.
---

# MCP - Model Context Protocol client

## Overview

`ov test mcp` connects to Model Context Protocol servers declared by running containers via `mcp_provides`, using [github.com/modelcontextprotocol/go-sdk](https://pkg.go.dev/github.com/modelcontextprotocol/go-sdk) (v1.5.0). Seven leaf subcommands cover the full MCP client surface: `ping`, `servers`, `list-tools`, `list-resources`, `list-prompts`, `call`, `read`. No MCP URL argument is ever typed by the user — the verb reads the target image's `org.overthinkos.mcp_provides` OCI label, resolves `{{.ContainerName}}` templates, applies pod-aware `localhost` rewriting, and maps the container-network URL to the published host port automatically.

### Also as a declarative verb

Every `ov test mcp <method>` is authorable as an `mcp:` verb inside a `tests:` block. The method name becomes the verb's YAML value; method-specific args are sibling fields (`tool:`, `uri:`, `input:`, `mcp_name:`). Shared matchers (`stdout:`, `stderr:`, `exit_status:`, `timeout:`) work like other verbs. See `/ov:test` for the parent router and the complete method allowlist. Example:

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
| Ping | `ov test mcp ping <image>` | Liveness check — returns `ok` on a successful `Ping` RPC |
| Enumerate | `ov test mcp servers <image>` | List MCP servers declared by the image (no dial) |
| List tools | `ov test mcp list-tools <image>` | Tool name + first-line description per line |
| List resources | `ov test mcp list-resources <image>` | URI + name + MIME type per line |
| List prompts | `ov test mcp list-prompts <image>` | Prompt name + description per line |
| Call | `ov test mcp call <image> <tool> [<input-json>]` | Invoke a tool; prints TextContent payload |
| Read | `ov test mcp read <image> <uri>` | Read a resource; prints Text content |

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
ov test mcp ping jupyter
# ok
```

Calls `ClientSession.Ping(ctx, nil)`. Exit 0 iff the server responds. Useful as a liveness check, often paired with a short `--timeout` in CI.

### Servers (discovery only, no dial)

```bash
ov test mcp servers jupyter
# jupyter   http://localhost:8888/mcp   http

ov test mcp servers jupyter --json
# [{"name":"jupyter","url":"http://localhost:8888/mcp","transport":"http","source":"jupyter"}]
```

Reads `mcp_provides` from the OCI label, applies template substitution + pod-aware rewrite, prints the result. No MCP handshake. Useful for confirming which server names the image advertises before dialing.

### List tools / resources / prompts

```bash
ov test mcp list-tools jupyter
# list_notebooks       List all notebooks accessible in the workspace.
# execute_cell         Execute a cell and return its outputs.
# insert_cell          Insert a new cell at the given position.
# …

ov test mcp list-tools jupyter --json | jq '.[].name'
```

Plaintext output is tab-separated (`name\tfirst-line-of-description`) for easy `contains:` matching in declarative tests without JSON parsing. Multi-line descriptions collapse to the first non-empty line. `list-resources` emits `uri\tname\tmime`; `list-prompts` emits `name\tdescription`. The runner automatically pages through `NextCursor` so all results return in a single invocation.

### Call a tool

```bash
ov test mcp call jupyter list_notebooks '{}'
# [{"path":"finetuning/01_FastInference_Qwen.ipynb",…}]

ov test mcp call jupyter get_cell '{"path":"intro.ipynb","index":0}'
```

The third positional is the tool's arguments as a JSON object (optional — omit for zero-arg tools). The argument string is parsed with `encoding/json`; parse errors exit 1 immediately with the error location. Returned `TextContent` blocks print one per line; `ImageContent` / `AudioContent` emit a `[image content: <mime>, <N> bytes]` placeholder (use `--json` for the actual payload). If the server sets `IsError: true`, the leaf exits 1 after printing the error text to stderr.

### Read a resource

```bash
ov test mcp read my-image file:///workspace/data.txt
```

Calls `ReadResource(URI: …)`. Text content is printed to stdout; binary blobs emit a `[binary resource … use --json for base64]` placeholder to stderr.

## Disambiguating multiple servers

When an image declares more than one `mcp_provides` entry, every leaf requires `--name`:

```bash
ov test mcp ping my-image
# ov: error: image provides multiple mcp servers; use --name (available: jupyter, chrome-devtools)

ov test mcp ping my-image --name chrome-devtools
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

**Symptom:** `ov test mcp ping` fails with:

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
ov test mcp ping <image>
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

`--json` on any leaf emits the SDK's raw result struct (e.g., `ListToolsResult`, `CallToolResult`) with `json.MarshalIndent`. Declarative tests always receive plaintext — the subprocess dispatcher runs `ov test mcp …` without `--json`.

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

**Deploy-scope only.** `mcp:` checks require a running container with the mcp port published; `ov image validate` rejects build-scope mcp checks at authoring time, and `ov image test` skips them at runtime with the message `"mcp: <method> requires a running container (skip under ov image test)"`. Follow the same rule as the other four live-container verbs — `cdp`, `wl`, `dbus`, `vnc`.

## Validator coverage

`ov image validate` enforces at the `validateOvVerb` dispatch in `ov/validate_tests.go`:

- Method name must be in `mcpMethods` (7 entries); unknown methods list the allowed set in the error.
- `scope` must be `deploy`; build-scope raises `"mcp: verb requires scope:\"deploy\""`.
- `call` requires `tool:`; missing field raises `"mcp: call requires modifier \"tool\""`.
- `read` requires `uri:`.

No other required modifiers — `ping`, `servers`, `list-*` take only the optional `mcp_name:`.

## Cross-References

- `/ov:test` — parent router; all `ov test mcp …` invocations are dispatched through it. Full method allowlist for all 5 live-container verbs (cdp/wl/dbus/vnc/mcp) lives in the "Live-container verb catalog" section.
- `/ov:layer` — `mcp_provides` / `mcp_accepts` / `mcp_requires` field reference for layer authoring.
- `/ov:config` — how `mcp_provides` gets injected into `deploy.yml` `provides.mcp:` and synthesized into `OV_MCP_SERVERS` for consumers at `ov config` time; pod-aware resolution to `localhost`; instance-aware MCP server naming with `-<instance>` suffix.
- `/ov:validate` — authoring-time validation rules (method allowlist, required modifiers, scope enforcement).
- `/ov:deploy` — `deploy.yml` `ports:` override semantics (the port-publishing gotcha lives here operationally).
- `/ov:cdp` — sibling live-container verb (Chrome DevTools Protocol).
- `/ov:wl` — sibling (Wayland desktop control).
- `/ov:dbus` — sibling (D-Bus calls/notifications).
- `/ov:vnc` — sibling (VNC framebuffer / input).
- `/ov-layers:jupyter-mcp` — the FastMCP server implementation layer (13 tools for notebook manipulation over CRDT).
- `/ov-layers:chrome-devtools-mcp` — the mcp-proxy wrapper around chrome-devtools-mcp (29 tools for browser automation).
- `/ov-layers:hermes` — a consumer (`mcp_accepts: jupyter, chrome-devtools`); use `ov test mcp` to verify the services hermes discovers are actually alive.
- `/ov-layers:openwebui` — another consumer (`mcp_accepts: jupyter, chrome-devtools`).
- `/ov-images:jupyter`, `/ov-images:jupyter-ml`, `/ov-images:jupyter-ml-notebook` — images bundling `jupyter-mcp`; `ov test <image> --filter mcp` exercises the verb end-to-end.
- `/ov-images:sway-browser-vnc`, `/ov-images:selkies-desktop`, `/ov-images:selkies-desktop-nvidia` — images bundling `chrome-devtools-mcp` (transitively via the chrome metalayer).
- `/ov-dev:go` — implementation map: `mcp.go` (Kong subcommand tree), `mcp_client.go` (SDK wrapper + URL rewriter), `testrun_ov_verbs.go` (declarative dispatcher entry `mcpMethods`), `validate_tests.go` (`validateOvVerb` case for `mcp`).

## When to Use This Skill

**MUST be invoked** when the task involves Model Context Protocol clients, `ov test mcp` commands, probing or testing MCP servers declared via `mcp_provides`, examining MCP tool/resource/prompt catalogs, debugging the URL-rewriter or port-publishing behavior, or authoring deploy-scope `mcp:` checks in a `tests:` block. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Test mode, live-container phase. Deploy an image with `mcp_provides`, start the service, then use `ov test mcp` to verify the endpoint speaks MCP and exposes the expected tools. Pairs naturally with `/ov:test` (for the parent router + declarative-verb catalog), `/ov:config` (for consumer-side injection), and the consumer-layer skills (`/ov-layers:hermes`, `/ov-layers:openwebui`).
