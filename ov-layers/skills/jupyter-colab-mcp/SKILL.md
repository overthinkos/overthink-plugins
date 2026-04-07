---
name: jupyter-colab-mcp
description: |
  JupyterLab CRDT MCP server extension with 13 tools for programmatic notebook access.
  MUST be invoked when working with: the MCP server implementation, CRDT collaboration,
  or the Tier 1 pip-only installation pattern for jupyter extensions.
---

# jupyter-colab-mcp -- JupyterLab CRDT MCP server extension

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | *(none)* |
| Packages | *(none -- pip-only Tier 1 layer)* |
| Services | *(none)* |
| Volumes | *(none)* |
| Install files | `layer.yml`, `user.yml`, `jupyter_colab_mcp/` (Python package) |

## Architecture: Tier 1 Post-Install Layer

This is a **Tier 1 "post-install"** layer — it has no `pixi.toml` and installs into whatever pixi environment exists from the parent Tier 2 layer. It follows the same pattern as `llama-cpp` and `unsloth`.

**Single source of truth:** The `jupyter_colab_mcp` Python package lives only in this layer. Both `jupyter-colab` (lightweight) and `jupyter-colab-ml` (GPU ML) compose it via their `layers:` field. This prevents code duplication and ensures bug fixes propagate to all images.

## How It Works

The `user.yml` performs three operations:

1. **Install FastMCP** — `pip install "fastmcp>=3.2.0"` (not via pixi because pixi's cross-platform resolver conflicts with opentelemetry-api on aarch64)
2. **Install jupyter-colab-mcp extension** — `pip install --no-deps /ctx/jupyter_colab_mcp` (from the layer's build context)
3. **Enable Jupyter Server extension** — writes `jupyter_colab_mcp.json` to the Jupyter server config directory

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
layers/jupyter-colab-mcp/
  layer.yml              # Description only (Tier 1, no deps)
  user.yml               # fastmcp + pip install + extension enable
  jupyter_colab_mcp/     # Python package
    pyproject.toml
    jupyter_colab_mcp/
      __init__.py
      app.py             # Jupyter server extension entry point
      mcp_server.py      # FastMCP tool definitions (13 tools)
      rtc_adapter.py     # CRDT room management, kernel execution
      tornado_asgi.py    # Tornado-to-ASGI bridge for FastMCP
```

## Integration with mcp_provides

The parent `jupyter-colab` layer declares `mcp_provides` to make this MCP server discoverable to other services at deploy time. The hermes service auto-discovers this server via the `OV_MCP_SERVERS` env var and registers all 13 tools as `mcp_jupyter_colab_<tool_name>`.

## Used In Layers

- `/ov-layers:jupyter-colab` — lightweight multi-arch JupyterLab (`layers: [jupyter-colab-mcp]`)
- `/ov-layers:jupyter-colab-ml` — GPU ML JupyterLab (`layers: [llama-cpp, unsloth, jupyter-colab-mcp]`)

## Used In Images

- `/ov-images:jupyter-colab`
- `/ov-images:jupyter-colab-ml`
- `/ov-images:jupyter-colab-ml-notebook`

## Related Skills

- `/ov-layers:jupyter-colab` — lightweight Tier 2 parent layer
- `/ov-layers:jupyter-colab-ml` — GPU ML Tier 2 parent layer
- `/ov:layer` — layer authoring rules (Tier 1 pattern)

## When to Use This Skill

Use when the user asks about:

- The jupyter-colab-mcp layer or its MCP server implementation
- How the CRDT collaboration works in JupyterLab
- The 13 MCP tools for notebook manipulation
- How the MCP extension is installed and enabled
- The Tier 1 extraction pattern (avoiding code duplication across Tier 2 layers)
