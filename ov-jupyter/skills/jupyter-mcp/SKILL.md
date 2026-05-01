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

The extension registers a Streamable HTTP MCP server at `http://localhost:8888/mcp` (MCP spec 2025-11-25). It provides 15 tools for programmatic notebook access, all named in `<noun>_<verb>` form.

**Naming convention.** Three nouns partition the catalog:
- `notebook_*` — filesystem operations on `.ipynb` files
- `cell_*` — in-memory cell mutations (require an open room)
- `room_*` — CRDT room lifecycle and introspection

**Room creation is ALWAYS explicit.** Only `room_open` creates a room.
Every room-mutation tool (`cell_*`, `notebook_get`, `notebook_watch`)
raises `RoomNotOpenError` when no room exists for the path — call
`room_open(path)` first, then `room_close(path)` when done.

| Tool | Description |
|------|-------------|
| `notebook_list` | List all notebooks in the workspace (filesystem) |
| `notebook_create` | Create a new empty notebook on disk (filesystem; does NOT open a room) |
| `notebook_get` | Get full notebook content from the live CRDT document; hard-fails with `RoomNotOpenError` if no room |
| `notebook_watch` | Block until a CRDT change is observed; hard-fails if no room |
| `cell_get` | Get a specific cell's content; hard-fails if no room |
| `cell_update` | Replace a cell's source; hard-fails if no room |
| `cell_insert` | Insert a new cell at a position; hard-fails if no room |
| `cell_delete` | Delete a cell; hard-fails if no room |
| `cell_execute` | Execute a cell, return outputs, AND write outputs + `execution_count` back via the CRDT-aware `set_cell` path; hard-fails if no room |
| `room_open` | Open or join a CRDT room (idempotent). Subsequent JupyterLab UI tabs that open the same notebook auto-join this same room |
| `room_close` | Synchronously flush pending CRDT state to disk, then evict the room; hard-fails if no room |
| `room_close_all` | Blanket cleanup — close every active room; returns `{closed: [...], errors: [...]}` |
| `room_pick` | Look up an existing room without creating one; hard-fails if no room |
| `room_list` | List every active CRDT room with rich metadata per entry: `room_id`, `path`, `file_id`, `users`, `user_count`, `has_kernel` |
| `room_list_users` | List awareness users currently connected to any room |

### CRDT Collaboration

Cell operations mutate the live CRDT document — changes appear instantly in all connected JupyterLab clients. Multiple MCP clients and browser users can edit the same notebook simultaneously. The server uses `jupyter-server-ydoc` for CRDT document management.

**Key implementation details (2026-05-01):**

1. **Room initialization on open.** `_create_room()` calls `await room.initialize()` after `server.start_room(room)` to load notebook content from disk into the CRDT document. Without this, the YNotebook is empty (0 cells). Mirrors `YDocWebSocketHandler.open()` in `jupyter-server-ydoc`.

2. **`cell_execute` persists outputs.** After collecting iopub messages, the adapter translates them via `nbformat.v4.output_from_msg`, captures `execution_count` from the shell-channel `execute_reply` (works for `print()`/`display()`-only cells that don't emit `execute_result`), and writes `cell["outputs"]` + `cell["execution_count"]` back via `set_cell()` — same CRDT-aware path used by `cell_update`. The MCP-level return path is unchanged. Without this, outputs only land on disk if `jupyter-collaboration`'s iopub→CRDT auto-bridge happened to fire in time (racy across many cells).

3. **`room_close` flushes synchronously, no defensive pop.** Calls `room._save_to_disc()` and awaits the returned task BEFORE `server.delete_room()`. Required because `DocumentRoom.stop()` (invoked by `delete_room`) cancels `self._saving_document` rather than awaiting it — any CRDT mutations sitting in the `save_delay` debounce would otherwise be lost. Wraps `server.delete_room` in try/except for `ValueError("not in list")` because the upstream `YDocWebSocketHandler._clean_room` may have already cleaned up the room when a UI client disconnects concurrently. The pre-cutover code did a defensive `server.rooms.pop()` after `delete_room` — that line was the trigger for the upstream race and has been removed.

### Persistence contract

`cell_execute → room_close → on-disk .ipynb` guarantees:
- Every executed cell's `outputs` array reflects the kernel's iopub stream (text, display_data, execute_result, error).
- Every executed cell's `execution_count` is a positive int.
- After `room_close` returns, `room_list` no longer reports the path's room.

This contract is locked by the eval.yml recipe `jupyter-execute-cell-persistence` (4 scenarios: single-cell print, parallel multi-cell, rich HTML display_data, room-eviction-after-close).

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

- `/ov-jupyter:jupyter` — lightweight multi-arch JupyterLab (`layers: [jupyter-mcp]`)
- `/ov-jupyter:jupyter-ml` — GPU ML JupyterLab (`layers: [llama-cpp, unsloth, jupyter-mcp]`)

## Used In Images

- `/ov-jupyter:jupyter`
- `/ov-jupyter:jupyter-ml`
- `/ov-jupyter:jupyter-ml-notebook`

## Related Skills

- `/ov-jupyter:jupyter` — lightweight Tier 2 parent layer
- `/ov-jupyter:jupyter-ml` — GPU ML Tier 2 parent layer
- `/ov-build:layer` — layer authoring rules (Tier 1 pattern)
- `/ov-build:mcp` — client-side verb for probing this server's tool catalog (ping, list-tools, call); use `ov eval mcp list-tools jupyter` to see all 13 tools this layer registers
- `/ov-selkies:chrome-devtools-mcp` — sibling MCP-server-provider layer for Chrome DevTools (different domain, same `mcp_provides` pattern)
- `/ov-hermes:hermes` — downstream MCP consumer (auto-discovers `jupyter` via `OV_MCP_SERVERS`; uses the 13 tools to read/edit/execute notebook cells programmatically)
- `/ov-openwebui:openwebui` — downstream MCP consumer (sets `CODE_EXECUTION_ENGINE=jupyter` when this server is discovered, routing Open WebUI's in-chat code blocks to the Jupyter kernel)

## When to Use This Skill

Use when the user asks about:

- The jupyter-mcp layer or its MCP server implementation
- How the CRDT collaboration works in JupyterLab
- The 13 MCP tools for notebook manipulation
- How the MCP extension is installed and enabled
- The Tier 1 extraction pattern (avoiding code duplication across Tier 2 layers)

## Related

- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
