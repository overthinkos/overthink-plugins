---
name: jupyter
description: |
  Lightweight JupyterLab with real-time collaboration on port 8888. No GPU required.
  Based on fedora (not nvidia), supports both amd64 and arm64.
  MUST be invoked before building, deploying, configuring, or troubleshooting the jupyter image.
---

# jupyter

Lightweight JupyterLab with real-time collaboration via jupyter-collaboration (Y-CRDT).

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, jupyter (sub-layers: jupyter-mcp), notebook-templates, dbus, ov |
| Platforms | linux/amd64, linux/arm64 |
| Ports | 8888 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (Fedora 43 base — no GPU)
2. `pixi` → `python` → `supervisord` (transitive)
3. `jupyter` — JupyterLab + jupyter-collaboration + data science (composes `jupyter-mcp` sub-layer for MCP extension)
4. `notebook-templates` — Starter notebooks (data layer, seeds ~/workspace)
5. `agent-forwarding` — SSH/GPG agent forwarding
5. `dbus` — D-Bus session bus
6. `ov` — ov CLI binary

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 8888 | JupyterLab | HTTP |

## Volumes

| Name | Path | Purpose |
|------|------|---------|
| workspace | ~/workspace | Persistent notebook storage |

## Quick Start

```bash
ov image build jupyter
ov config jupyter
ov start jupyter
# Open http://localhost:8888
```

## Key Layers

- `/ov-layers:jupyter` — JupyterLab + collaboration + data science
- `/ov-layers:agent-forwarding` — SSH/GPG forwarding

## Related Images

- `/ov-images:jupyter-ml` — GPU-accelerated variant (nvidia base, full ML stack)
- `/ov-images:jupyter-ml-notebook` — GPU variant with fine-tuning notebooks
- `/ov-images:openwebui` — Open WebUI consumes jupyter for code execution and MCP tools
- `/ov-images:hermes` — Hermes agent consumes jupyter MCP for notebook manipulation
- `/ov-images:fedora` — parent base image

## MCP Server (Programmatic Notebook Access)

The image includes a built-in MCP server at `http://localhost:8888/mcp` (Streamable HTTP transport, MCP spec 2025-11-25). AI agents can create, read, edit, execute, and watch notebooks programmatically — with changes syncing live to all collaborators via CRDT.

The MCP server name is set via the `MCP_SERVER_NAME` environment variable (default: `jupyter`). For multi-instance deployments, override per-instance: `ov config jupyter -i work -e MCP_SERVER_NAME=jupyter-work`.

### Configure Claude Code

The `ov-jupyter` plugin in `plugins/ov-jupyter/` declares the MCP server. To connect manually:

```bash
claude mcp add --transport http --scope project jupyter http://localhost:8888/mcp
```

This creates `.mcp.json` at the project root:
```json
{"mcpServers": {"jupyter": {"type": "http", "url": "http://localhost:8888/mcp"}}}
```

### 13 MCP tools

| Category | Tools |
|----------|-------|
| Notebook management | `list_notebooks`, `get_notebook`, `create_notebook` |
| Cell operations (CRDT) | `get_cell`, `update_cell`, `insert_cell`, `delete_cell`, `execute_cell` |
| Room management | `open_notebook_session`, `close_notebook_session` |
| Change watching | `watch_notebook` (blocks until change or timeout) |
| Collaboration | `get_active_users`, `get_active_sessions` |

See `/ov-layers:jupyter` for full parameter and return type documentation.

### Testing with `claude -p`

```bash
# Prerequisites: ov start jupyter

# Create and work with a notebook
claude -p "Call create_notebook with path 'test.ipynb'"
claude -p "Call open_notebook_session with path 'test.ipynb'"
claude -p "Call insert_cell with path 'test.ipynb', index 0, source 'print(42)', cell_type 'code'"
claude -p "Call execute_cell with path 'test.ipynb' and index 0"
claude -p "Call close_notebook_session with path 'test.ipynb'"
```

### Multi-client collaboration

Multiple `claude -p` sessions (or any MCP client) can edit the same notebook simultaneously. Each session creates a separate HTTP connection but shares the same CRDT document. Changes from one client are immediately visible to all others.

```bash
# Client A watches for changes
claude -p "Call watch_notebook with path 'test.ipynb' and timeout 30" &

# Client B makes a change
claude -p "Call update_cell with path 'test.ipynb', index 0, source 'print(\"hello\")'"

# Client A's watch_notebook returns: {"changed": true, "cell_count": 1}
```

## Verification

After `ov start`:

```bash
# Container and services running
ov status jupyter
ov service status jupyter

# JupyterLab responds
curl -s -o /dev/null -w '%{http_code}' http://localhost:8888    # 200

# MCP server responds
curl -s http://localhost:8888/mcp -X POST \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"test","version":"0.1.0"}}}'
# Expected: SSE response with serverInfo.name = "jupyter"

# Server extensions
ov shell jupyter -c "jupyter server extension list 2>&1 | grep -E 'ydoc|colab_mcp'"
# Expected: jupyter_server_ydoc enabled OK, jupyter_mcp 0.1.0 OK

# MCP tools via Claude Code
claude mcp list    # Should show: jupyter: http://localhost:8888/mcp (HTTP) - ✓ Connected
```

## Testing Collaboration

To test real-time collaboration, deploy `sway-browser-vnc` alongside:

```bash
ov start sway-browser-vnc
# Open JupyterLab in two Chrome tabs via container DNS:
ov cdp open sway-browser-vnc "http://ov-jupyter:8888/lab"
# Open second tab
ov cdp open sway-browser-vnc "http://ov-jupyter:8888/lab"
```

**Executing cells via CDP:** Use `Input.dispatchKeyEvent` (not VNC keys — unreliable when Chrome lacks compositor focus):

```bash
TAB=<tab-id>
# Focus cell, then Shift+Enter via CDP
ov cdp eval sway-browser-vnc $TAB "document.querySelector('.jp-Cell-inputArea .cm-content')?.focus()"
ov cdp raw sway-browser-vnc $TAB 'Input.dispatchKeyEvent' '{"type":"rawKeyDown","windowsVirtualKeyCode":13,"nativeVirtualKeyCode":13,"modifiers":8}'
ov cdp raw sway-browser-vnc $TAB 'Input.dispatchKeyEvent' '{"type":"keyUp","windowsVirtualKeyCode":13,"nativeVirtualKeyCode":13,"modifiers":8}'
```

## Test Coverage

Latest `ov test jupyter` run: **29 passed, 0 failed, 0 skipped**.
All tests embedded in the `org.overthinkos.tests` OCI label:
jupyter-lab binary under pixi, notebook-templates provisioned into
`${HOME}/workspace`, jupyter-mcp extension enabled, fastmcp pip
package installed. Deploy-scope: supervisord up, port 8888 reachable
on `127.0.0.1`, `/api` returns 200 with `version` in body, `/mcp`
returns 400 on empty POST (proving MCP routing is wired). Image-scope:
`jupyter_mcp` appears in extension list, workspace has ≥1 `.ipynb`.

See `/ov:test` for the framework and author-facing gotchas.

## Related Skills

- `/ov-layers:jupyter`, `/ov-layers:jupyter-mcp`, `/ov-layers:notebook-templates`
- `/ov:test` — declarative testing framework
- `/ov:config` — deploy setup
- `/ov-images:jupyter-ml`, `/ov-images:jupyter-ml-notebook` — GPU variants

## When to Use This Skill

**MUST be invoked** when the task involves the jupyter image, collaborative Jupyter notebooks, lightweight Jupyter deployments without GPU, MCP-based notebook access, or multi-client collaboration. Invoke this skill BEFORE reading source code or launching Explore agents.
