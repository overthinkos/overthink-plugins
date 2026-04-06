---
name: jupyter-colab
description: |
  Lightweight JupyterLab with real-time collaboration on port 8888. No GPU required.
  Based on fedora (not nvidia), supports both amd64 and arm64.
  MUST be invoked before building, deploying, configuring, or troubleshooting the jupyter-colab image.
---

# jupyter-colab

Lightweight JupyterLab with real-time collaboration via jupyter-collaboration (Y-CRDT).

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, jupyter-colab (sub-layers: jupyter-colab-mcp), dbus, ov |
| Platforms | linux/amd64, linux/arm64 |
| Ports | 8888 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (Fedora 43 base — no GPU)
2. `pixi` → `python` → `supervisord` (transitive)
3. `jupyter-colab` — JupyterLab + jupyter-collaboration + data science (composes `jupyter-colab-mcp` sub-layer for MCP extension)
4. `agent-forwarding` — SSH/GPG agent forwarding
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
ov build jupyter-colab
ov config jupyter-colab
ov start jupyter-colab
# Open http://localhost:8888
```

## Key Layers

- `/ov-layers:jupyter-colab` — JupyterLab + collaboration + data science
- `/ov-layers:agent-forwarding` — SSH/GPG forwarding

## Related Images

- `/ov-images:jupyter` — GPU-accelerated variant (nvidia base, ML libraries, amd64 only)
- `/ov-images:fedora` — parent base image

## MCP Server (Programmatic Notebook Access)

The image includes a built-in MCP server at `http://localhost:8888/mcp` (Streamable HTTP transport, MCP spec 2025-11-25). AI agents can create, read, edit, execute, and watch notebooks programmatically — with changes syncing live to all collaborators via CRDT.

### Configure Claude Code

The `ov-jupyter` plugin in `plugins/ov-jupyter/` declares the MCP server. To connect manually:

```bash
claude mcp add --transport http --scope project jupyter-colab http://localhost:8888/mcp
```

This creates `.mcp.json` at the project root:
```json
{"mcpServers": {"jupyter-colab": {"type": "http", "url": "http://localhost:8888/mcp"}}}
```

### 13 MCP tools

| Category | Tools |
|----------|-------|
| Notebook management | `list_notebooks`, `get_notebook`, `create_notebook` |
| Cell operations (CRDT) | `get_cell`, `update_cell`, `insert_cell`, `delete_cell`, `execute_cell` |
| Room management | `open_notebook_session`, `close_notebook_session` |
| Change watching | `watch_notebook` (blocks until change or timeout) |
| Collaboration | `get_active_users`, `get_active_sessions` |

See `/ov-layers:jupyter-colab` for full parameter and return type documentation.

### Testing with `claude -p`

```bash
# Prerequisites: ov start jupyter-colab

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
ov status jupyter-colab
ov service status jupyter-colab

# JupyterLab responds
curl -s -o /dev/null -w '%{http_code}' http://localhost:8888    # 200

# MCP server responds
curl -s http://localhost:8888/mcp -X POST \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"test","version":"0.1.0"}}}'
# Expected: SSE response with serverInfo.name = "jupyter-colab"

# Server extensions
ov shell jupyter-colab -c "jupyter server extension list 2>&1 | grep -E 'ydoc|colab_mcp'"
# Expected: jupyter_server_ydoc enabled OK, jupyter_colab_mcp 0.1.0 OK

# MCP tools via Claude Code
claude mcp list    # Should show: jupyter-colab: http://localhost:8888/mcp (HTTP) - ✓ Connected
```

## Testing Collaboration

To test real-time collaboration, deploy `sway-browser-vnc` alongside:

```bash
ov start sway-browser-vnc
# Open JupyterLab in two Chrome tabs via container DNS:
ov cdp open sway-browser-vnc "http://ov-jupyter-colab:8888/lab"
# Open second tab
ov cdp open sway-browser-vnc "http://ov-jupyter-colab:8888/lab"
```

**Executing cells via CDP:** Use `Input.dispatchKeyEvent` (not VNC keys — unreliable when Chrome lacks compositor focus):

```bash
TAB=<tab-id>
# Focus cell, then Shift+Enter via CDP
ov cdp eval sway-browser-vnc $TAB "document.querySelector('.jp-Cell-inputArea .cm-content')?.focus()"
ov cdp raw sway-browser-vnc $TAB 'Input.dispatchKeyEvent' '{"type":"rawKeyDown","windowsVirtualKeyCode":13,"nativeVirtualKeyCode":13,"modifiers":8}'
ov cdp raw sway-browser-vnc $TAB 'Input.dispatchKeyEvent' '{"type":"keyUp","windowsVirtualKeyCode":13,"nativeVirtualKeyCode":13,"modifiers":8}'
```

## Differences from jupyter Image

| | jupyter-colab | jupyter |
|---|---|---|
| Base | fedora | nvidia |
| GPU | None | CUDA |
| Platforms | amd64 + arm64 | amd64 only |
| Focus | Collaboration + MCP | ML/AI training |
| MCP server | Yes (13 tools) | Yes (via jupyter-mcp-server) |
| Size | ~3.4 GB | ~15+ GB |

## When to Use This Skill

**MUST be invoked** when the task involves the jupyter-colab image, collaborative Jupyter notebooks, lightweight Jupyter deployments without GPU, MCP-based notebook access, or multi-client collaboration. Invoke this skill BEFORE reading source code or launching Explore agents.
