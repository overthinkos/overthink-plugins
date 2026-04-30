---
name: jupyter-mcp
description: |
  JupyterLab CRDT MCP server extension with 13 tools for programmatic notebook access.
  MUST be invoked when working with: the MCP server implementation, CRDT collaboration,
  or the Tier 1 pip-only installation pattern for jupyter extensions.
---

# jupyter-mcp -- JupyterLab CRDT MCP server extension

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | *(none)* |
| Packages | *(none -- pip-only Tier 1 layer)* |
| Services | *(none)* |
| Volumes | *(none)* |
| Install files | `layer.yml`, `tasks:`, `jupyter_mcp/` (Python package) |

## Architecture: Tier 1 Post-Install Layer

This is a **Tier 1 "post-install"** layer — it has no `pixi.toml` and installs into whatever pixi environment exists from the parent Tier 2 layer. It follows the same pattern as `llama-cpp` and `unsloth`.

**Single source of truth:** The `jupyter_mcp` Python package lives only in this layer. Both `jupyter` (lightweight) and `jupyter-ml` (GPU ML) compose it via their `layers:` field. This prevents code duplication and ensures bug fixes propagate to all images.

## How It Works

The `tasks:` performs three operations:

1. **Install FastMCP** — `pip install "fastmcp>=3.2.0"` (not via pixi because pixi's cross-platform resolver conflicts with opentelemetry-api on aarch64)
2. **Install jupyter-mcp extension** — `pip install --no-deps /ctx/jupyter_mcp` (from the layer's build context)
3. **Enable Jupyter Server extension** — writes `jupyter_mcp.json` to the Jupyter server config directory

## MCP Server

The extension registers a Streamable HTTP MCP server at `http://localhost:8888/mcp` (MCP spec 2025-11-25). It provides 13 tools for programmatic notebook access:

| Tool | Description |
|------|-------------|
| `list_notebooks` | List all notebooks in the workspace |
| `get_notebook` | Get full notebook content (cells, metadata) |
| `create_notebook` | Create a new notebook |
| `open_notebook_session` | Open a CRDT collaboration room |
| `close_notebook_session` | Close a CRDT room and save to disk |
| `get_cell` | Get a specific cell's content |
| `update_cell` | Replace a cell's source |
| `insert_cell` | Insert a new cell at a position |
| `delete_cell` | Delete a cell |
| `execute_cell` | Execute a cell and return outputs |
| `get_active_sessions` | List active collaboration rooms |
| `get_active_users` | List connected users |
| `watch_notebook` | Watch for real-time changes |

### CRDT Collaboration

Cell operations mutate the live CRDT document — changes appear instantly in all connected JupyterLab clients. Multiple MCP clients and browser users can edit the same notebook simultaneously. The server uses `jupyter-server-ydoc` for CRDT document management.

**Key implementation detail:** The `_create_room()` method calls `await room.initialize()` after `server.start_room(room)` to load notebook content from disk into the CRDT document. Without this, the YNotebook is empty (0 cells). This mirrors the `YDocWebSocketHandler.open()` behavior in `jupyter-server-ydoc`.

### Architecture Stack

```
Claude Code / MCP Client
    ↓ Streamable HTTP (POST /mcp)
FastMCP Server (mcp_server.py)
    ↓
JupyterLab RTC Adapter (rtc_adapter.py)
    ↓ CRDT operations
jupyter-server-ydoc DocumentRoom
    ↓ Y.js document sync
JupyterLab Kernel Manager (execute_cell)
```

## Key Files

```
layers/jupyter-mcp/
  layer.yml              # Description only (Tier 1, no deps)
  # tasks: block in layer.yml — fastmcp + pip install + extension enable
  jupyter_mcp/           # Python package
    pyproject.toml
    jupyter_mcp/
      __init__.py
      app.py             # Jupyter server extension entry point
      mcp_server.py      # FastMCP tool definitions (13 tools)
      rtc_adapter.py     # CRDT room management, kernel execution
      tornado_asgi.py    # Tornado-to-ASGI bridge for FastMCP
```

## Integration with mcp_provides

The parent `jupyter` layer declares `mcp_provides` to make this MCP server discoverable to other services at deploy time. The hermes service auto-discovers this server via the `OV_MCP_SERVERS` env var and registers all 13 tools as `mcp_jupyter_<tool_name>`.

## MCP Name Decoupling (design principle)

The `jupyter` MCP server name is **deliberately decoupled** from the layer name, the Python package name, and the image name. Three places anchor it:

| File | Field | Value | Purpose |
|---|---|---|---|
| `layers/jupyter/layer.yml` | `env.MCP_SERVER_NAME` | `"jupyter"` | Runtime advertisement |
| `layers/jupyter/layer.yml` | `mcp_provides[0].name` | `jupyter` | Cross-container discovery (hermes, openwebui) |
| `plugins/ov-jupyter/.mcp.json` | `mcpServers.jupyter` | — | Claude Code static registration |

When the layer/package/image were renamed (`jupyter-colab-ml` → `jupyter-ml`, `jupyter_colab_mcp` → `jupyter_mcp`), these three values stayed `jupyter` — and every consumer (`hermes` `mcp_accepts: jupyter`, `openwebui` `mcp_accepts: jupyter`, the Claude Code plugin) continued working without any edits. **Package/layer/image names describe the artifact; the MCP name describes the service contract.** Rename the artifact freely; the contract is stable.

## Used In Layers

- `/ov-layers:jupyter` — lightweight multi-arch JupyterLab (`layers: [jupyter-mcp]`)
- `/ov-layers:jupyter-ml` — GPU ML JupyterLab (`layers: [llama-cpp, unsloth, jupyter-mcp]`)

## Used In Images

- `/ov-images:jupyter`
- `/ov-images:jupyter-ml`
- `/ov-images:jupyter-ml-notebook`

## Related Skills

- `/ov-layers:jupyter` — lightweight Tier 2 parent layer
- `/ov-layers:jupyter-ml` — GPU ML Tier 2 parent layer
- `/ov:layer` — layer authoring rules (Tier 1 pattern)
- `/ov:mcp` — client-side verb for probing this server's tool catalog (ping, list-tools, call); use `ov eval mcp list-tools jupyter` to see all 13 tools this layer registers
- `/ov-layers:chrome-devtools-mcp` — sibling MCP-server-provider layer for Chrome DevTools (different domain, same `mcp_provides` pattern)
- `/ov-layers:hermes` — downstream MCP consumer (auto-discovers `jupyter` via `OV_MCP_SERVERS`; uses the 13 tools to read/edit/execute notebook cells programmatically)
- `/ov-layers:openwebui` — downstream MCP consumer (sets `CODE_EXECUTION_ENGINE=jupyter` when this server is discovered, routing Open WebUI's in-chat code blocks to the Jupyter kernel)

## When to Use This Skill

Use when the user asks about:

- The jupyter-mcp layer or its MCP server implementation
- How the CRDT collaboration works in JupyterLab
- The 13 MCP tools for notebook manipulation
- How the MCP extension is installed and enabled
- The Tier 1 extraction pattern (avoiding code duplication across Tier 2 layers)

## Related

- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
