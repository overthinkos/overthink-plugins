---
name: jupyter-colab
description: |
  Lightweight JupyterLab with real-time collaboration (jupyter-collaboration) on port 8888.
  No GPU required. Use when working with collaborative notebooks, jupyter-collaboration,
  or lightweight Jupyter environments without ML/CUDA dependencies.
---

# jupyter-colab -- Lightweight JupyterLab with real-time collaboration

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord` |
| Sub-layers | `jupyter-colab-mcp` |
| Ports | 8888 |
| Service | `jupyter-colab` (supervisord) |
| Volume | `workspace` at `~/workspace` |
| Install files | `layer.yml` (rpm: git), `pixi.toml` |

## Key Packages

**conda-forge:** JupyterLab >= 4.4.0, ipywidgets, ipykernel, jupyterlab-git, jupyter-resource-usage, matplotlib, seaborn, pandas, numpy, scikit-learn, scipy, polars, pyarrow, duckdb, black, pytest

**PyPI:** jupyter-collaboration >= 4.1.0

**RPM:** git

## Environment

No custom environment variables. Inherits `PATH` from pixi layer (`~/.pixi/envs/default/bin`).

## Collaboration

Real-time collaborative editing via [jupyter-collaboration](https://github.com/jupyterlab/jupyter-collaboration) (Y-CRDT). Multiple users editing the same notebook see shared cursors and live changes. The backend uses `jupyter_server_ydoc` with `pycrdt-websocket` for CRDT synchronization. No external database required — document state is managed in-memory by default, with optional SQLite persistence.

Collaboration is **enabled by default** when jupyter-collaboration is installed. The WebSocket endpoint at `/api/collaboration/room/` handles document sync.

## Token Authentication

The service supports optional token-based authentication via the `JUPYTER_TOKEN_FILE` environment variable. If set and the file exists, its contents are used as the auth token. Otherwise, authentication is disabled (empty token, no password) for development convenience.

## Volume

The `workspace` volume is mounted at `~/workspace`. JupyterLab serves notebooks from this directory via `--notebook-dir=$HOME/workspace`.

## Usage

```yaml
# images.yml
jupyter-colab:
  base: fedora
  layers:
    - agent-forwarding
    - jupyter-colab
    - dbus
    - ov
  ports:
    - "8888:8888"
```

## Differences from jupyter Layer

| | jupyter-colab | jupyter |
|---|---|---|
| Base dependency | `supervisord` | `cuda`, `supervisord` |
| GPU | No | CUDA required |
| Platforms | amd64 + arm64 | amd64 only |
| Focus | Collaboration + data science | ML/AI training |
| Key packages | jupyter-collaboration, pandas, polars | PyTorch, vLLM, transformers, llama.cpp |
| Image size | ~3.4 GB | ~15+ GB |

## MCP Server Extension (`jupyter_colab_mcp`)

The layer includes a built-in MCP (Model Context Protocol) server at `/mcp` on port 8888. AI agents can create, read, edit, and execute notebook cells programmatically — with changes syncing live to all connected JupyterLab clients via CRDT.

### Architecture

```
Tornado (Jupyter Server, :8888)
  └── /mcp → TornadoASGIHandler → FastMCP ASGI App (Starlette, in-process)
                                      └── RTCAdapter → YNotebook (CRDT)
```

- **FastMCP v3.x** (standalone, by Prefect) with Streamable HTTP transport (MCP spec 2025-11-25)
- **Tornado-ASGI bridge** — custom handler translates Tornado requests to ASGI scope/receive/send (both share the same asyncio event loop)
- **On-demand room creation** — CRDT rooms are created headlessly when tools need them (no browser required). Replicates `YDocWebSocketHandler.prepare()` logic including lazy websocket server start
- **Per-notebook asyncio locks** — serializes mutations on the same notebook to prevent index-based race conditions during concurrent access
- **Lazy YDocExtension resolution** — resolved on first tool call, not at extension load time (avoids load-order dependency)

### MCP Tools (13 total)

**Notebook Management:**
| Tool | Parameters | Returns |
|------|-----------|---------|
| `list_notebooks` | — | `[{"path": "...", "name": "..."}]` |
| `get_notebook` | `path` | Full notebook dict (CRDT if room open, else disk) |
| `create_notebook` | `path` | `{"path": "...", "name": "..."}` |

**Cell Operations (CRDT — changes sync live to all collaborators):**
| Tool | Parameters | Returns |
|------|-----------|---------|
| `get_cell` | `path`, `index` | Cell dict (source, cell_type, metadata, outputs) |
| `update_cell` | `path`, `index`, `source`, `cell_type?` | `"Cell N updated"` |
| `insert_cell` | `path`, `index`, `source`, `cell_type="code"` | `"Cell inserted at index N"` |
| `delete_cell` | `path`, `index` | `"Cell N deleted"` |
| `execute_cell` | `path`, `index` | `[{"type": "stream", "content": {...}}]` |

**Room Management:**
| Tool | Parameters | Returns |
|------|-----------|---------|
| `open_notebook_session` | `path` | `"Collaboration room opened for ..."` |
| `close_notebook_session` | `path` | `"Collaboration room closed for ..."` |

**Change Watching:**
| Tool | Parameters | Returns |
|------|-----------|---------|
| `watch_notebook` | `path`, `timeout=30` | `{"changed": true, "cell_count": N}` or `{"changed": false}` |

**Collaboration Awareness:**
| Tool | Parameters | Returns |
|------|-----------|---------|
| `get_active_users` | — | `[{"id": "...", "name": "..."}]` |
| `get_active_sessions` | — | `[{"room_id": "..."}]` |

### Installation

The MCP extension is installed by the `jupyter-colab-mcp` sub-layer (composed via `layers: [jupyter-colab-mcp]`):
1. `pip install "fastmcp>=3.2.0"` + opentelemetry runtime deps (not pixi -- cross-platform resolver conflicts with opentelemetry-api on aarch64)
2. `pip install --no-deps /ctx/jupyter_colab_mcp` (extension package from sub-layer directory)
3. Extension enabled via `jupyter_server_config.d/jupyter_colab_mcp.json`

### Source files

```
layers/jupyter-colab-mcp/
├── jupyter_colab_mcp/
│   ├── pyproject.toml              # hatchling package, no deps (--no-deps install)
│   └── jupyter_colab_mcp/
│       ├── __init__.py             # Extension entry point
│       ├── app.py                  # Registers /mcp handler + ASGI lifespan
│       ├── tornado_asgi.py         # Tornado↔ASGI bridge (SSE streaming, disconnect handling)
│       ├── mcp_server.py           # FastMCP tool definitions (13 tools)
│       └── rtc_adapter.py          # CRDT access via YNotebook + kernel execution
├── user.yml                        # Build-time: pip install fastmcp + extension + enable
└── layer.yml
```

### Multi-client support

Multiple MCP clients can edit the same notebook simultaneously:
- All sessions share the same CRDT document via the singleton RTCAdapter
- Per-notebook asyncio locks prevent index corruption during concurrent mutations
- `watch_notebook` uses a fan-out `NotebookWatcher` — each watcher gets an independent `asyncio.Event`, signaled when any CRDT change occurs
- The CRDT layer (pycrdt) handles merge conflicts automatically

## Implementation Notes

- The supervisord service command calls `jupyter lab` directly, not via `pixi run`. Pixi multi-stage builds copy only `~/.pixi/envs/` to the final image — `pixi.toml` is not present at runtime.
- The `start-jupyter` pixi task in `pixi.toml` is for build-time verification only.

## Used In Images

- `/ov-images:jupyter-colab`

## Related Layers

- `/ov-layers:jupyter-colab-ml` -- GPU-accelerated variant with full CUDA ML stack + same CRDT MCP server
- `/ov-layers:jupyter` -- Legacy GPU Jupyter with ML libraries (monolithic, no CRDT MCP)
- `/ov-layers:supervisord` -- process manager dependency
- `/ov-layers:python` -- Python runtime (transitive via supervisord)

## When to Use This Skill

Use when the user asks about:

- Real-time collaborative Jupyter notebooks
- jupyter-collaboration or Y-CRDT setup
- Lightweight Jupyter without GPU/ML dependencies
- Port 8888 service (check if GPU variant intended)
- The `jupyter-colab` layer
- MCP server for notebook manipulation
- Programmatic notebook access from AI agents
- `watch_notebook` or change notifications
- Multi-client concurrent editing
