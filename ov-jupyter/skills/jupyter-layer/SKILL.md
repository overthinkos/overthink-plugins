---
name: jupyter
description: |
  Lightweight JupyterLab with real-time collaboration (jupyter-collaboration) on port 8888.
  No GPU required. Use when working with collaborative notebooks, jupyter-collaboration,
  or lightweight Jupyter environments without ML/CUDA dependencies.
---

# jupyter -- Lightweight JupyterLab with real-time collaboration

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord` |
| Sub-layers | `jupyter-mcp` |
| Ports | 8888 |
| Service | `jupyter` (supervisord) |
| Volume | `workspace` at `~/workspace` |
| MCP provides | `jupyter` at `http://{{.ContainerName}}:8888/mcp` (streamable HTTP) |
| Install files | `layer.yml` (rpm: git), `pixi.toml` |

## Key Packages

**conda-forge:** JupyterLab >= 4.4.0, ipywidgets, ipykernel, jupyterlab-git, jupyter-resource-usage, matplotlib, seaborn, pandas, numpy, scikit-learn, scipy, polars, pyarrow, duckdb, **spacy**, **spacy-model-en_core_web_sm**, black, pytest, graphviz, pyyaml, tqdm

**PyPI:** jupyter-collaboration >= 4.1.0

**RPM:** git

### NLP

`spacy` (3.8.x) ships with the small English statistical model
(`en_core_web_sm`) preloaded so notebooks can do tokenization, NER,
dependency parsing, and POS tagging out-of-the-box without a
post-deploy `python -m spacy download` step. Sample:

```python
import spacy
nlp = spacy.load("en_core_web_sm")
doc = nlp("Apple is looking at buying a U.K. startup for $1 billion")
[(t.text, t.pos_) for t in doc]   # tokens + parts of speech
[(e.text, e.label_) for e in doc.ents]  # named entities
```

The `spacy-import` build-scope eval check (in `layer.yml`) verifies the
package + model load successfully on every image build — a future
upstream rename or version bump that breaks the model load fails the
build loudly.

## Environment

No custom environment variables on the layer itself. **Runtime PATH
contributions** for the pixi env (`~/.pixi/bin`,
`~/.pixi/envs/default/bin`) and cache dirs (`PIXI_CACHE_DIR`,
`RATTLER_CACHE_DIR`) come from the **pixi BUILDER** (declared in
`build.yml` / `overthink.yml` under `builder.pixi.runtime_env:` and
`path_contributions:`) — see `/ov-dev:generate` for the
`writeLayerEnv` flow. This was moved from the pixi LAYER to the pixi
BUILDER on 2026-04-29 so images that consume pixi via
pixi.toml-triggered builds (jupyter, openwebui, immich-ml, …) get the
env contract automatically without needing pixi as a top-level layer
for sibling-grouped auto-intermediate inheritance.

## Collaboration

Real-time collaborative editing via [jupyter-collaboration](https://github.com/jupyterlab/jupyter-collaboration) (Y-CRDT). Multiple users editing the same notebook see shared cursors and live changes. The backend uses `jupyter_server_ydoc` with `pycrdt-websocket` for CRDT synchronization. No external database required — document state is managed in-memory by default, with optional SQLite persistence.

Collaboration is **enabled by default** when jupyter-collaboration is installed. The WebSocket endpoint at `/api/collaboration/room/` handles document sync.

## Token Authentication

The service supports optional token-based authentication via the `JUPYTER_TOKEN_FILE` environment variable. If set and the file exists, its contents are used as the auth token. Otherwise, authentication is disabled (empty token, no password) for development convenience.

## Volume

The `workspace` volume is mounted at `~/workspace`. JupyterLab serves notebooks from this directory via `--notebook-dir=$HOME/workspace`.

## Usage

```yaml
# image.yml
jupyter:
  base: fedora
  layers:
    - agent-forwarding
    - jupyter
    - dbus
    - ov
  ports:
    - "8888:8888"
```

## Differences from jupyter-ml Layer

| | jupyter | jupyter-ml |
|---|---|---|
| Base dependency | `supervisord` | `cuda`, `supervisord` |
| GPU | No | CUDA required |
| Platforms | amd64 + arm64 | amd64 only |
| Focus | Collaboration + data science | ML/AI training |
| Key packages | jupyter-collaboration, pandas, polars | PyTorch, vLLM, Unsloth, llama.cpp |
| Image size | ~3.4 GB | ~15+ GB |

## MCP Server Discovery

The `jupyter` layer declares `mcp_provides` for cross-container and pod MCP discovery. The URL template `http://{{.ContainerName}}:8888/mcp` resolves to the actual container name at deploy time (e.g., `http://ov-jupyter:8888/mcp`), or to `http://localhost:8888/mcp` in combined images where both services run in the same container.

- **Transport:** Streamable HTTP (`http`)
- **13 tools** available via the `jupyter-mcp` extension (see below)
- **Hermes auto-configures** via the `OV_MCP_SERVERS` env var -- no manual MCP registration needed
- **Pod-aware:** resolves to `localhost` in combined images where hermes and jupyter share a container

## MCP Server Extension (`jupyter_mcp`)

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
| `execute_cell` | `path`, `index` | `[{"type": "stream", "content": {...}}]` — also writes `outputs` + `execution_count` back to the cell via the CRDT path so they persist on `close_notebook_session` |

**Room Management:**
| Tool | Parameters | Returns |
|------|-----------|---------|
| `open_notebook_session` | `path` | `"Collaboration room opened for ..."` |
| `close_notebook_session` | `path` | `"Collaboration room closed for ..."` — synchronously flushes pending CRDT state to disk via `room._save_to_disc()` before evicting the room |

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

The MCP extension is installed by the `jupyter-mcp` sub-layer (composed via `layers: [jupyter-mcp]`):
1. `pip install "fastmcp>=3.2.0"` + opentelemetry runtime deps (not pixi -- cross-platform resolver conflicts with opentelemetry-api on aarch64)
2. `pip install --no-deps /ctx/jupyter_mcp` (extension package from sub-layer directory)
3. Extension enabled via `jupyter_server_config.d/jupyter_mcp.json`

### Source files

```
layers/jupyter-mcp/
├── jupyter_mcp/
│   ├── pyproject.toml              # hatchling package, no deps (--no-deps install)
│   └── jupyter_mcp/
│       ├── __init__.py             # Extension entry point
│       ├── app.py                  # Registers /mcp handler + ASGI lifespan
│       ├── tornado_asgi.py         # Tornado↔ASGI bridge (SSE streaming, disconnect handling)
│       ├── mcp_server.py           # FastMCP tool definitions (13 tools)
│       └── rtc_adapter.py          # CRDT access via YNotebook + kernel execution
# (in layer.yml tasks:)  # Build-time: pip install fastmcp + extension + enable
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

## Tests

The layer ships declarative checks embedded in the `org.overthinkos.eval`
OCI label (see `/ov-build:eval` for the full schema):

- **Build-scope** (run under `ov eval image`):
  - `jupyter-lab-binary` — `${HOME}/.pixi/envs/default/bin/jupyter-lab`
    exists (proves pixi env install succeeded)
  - `spacy-import` — `python -c "import spacy;
    spacy.load('en_core_web_sm')"` exits 0 (proves NLP packages + model
    load successfully)
- **Deploy-scope** (run under `ov test` against a live service):
  - `workspace-dir` — `${HOME}/workspace` exists (mount visible)
  - `jupyter-service` — supervisord program `jupyter` is RUNNING
  - `jupyter-port` — `${CONTAINER_IP}:${HOST_PORT:8888}` reachable
  - `jupyter-api` — `GET .../api` returns 200 with `"version"` in body
    (the `${HOST_PORT:8888}` substitution means the check works
    unchanged when `deploy.yml` remaps the host port)
  - `mcp-jupyter-ping` — MCP server responds to `ping`
  - `mcp-jupyter-list-tools` — assertion that all 13 documented MCP
    tools are present in `tools/list`
  - `mcp-jupyter-call-list-notebooks` — actually invokes
    `list_notebooks` and verifies the response shape

## Used In Images

- `/ov-jupyter:jupyter`

## Related Layers

- `/ov-jupyter:jupyter-ml` -- GPU-accelerated variant with full CUDA ML stack + same CRDT MCP server
- `/ov-jupyter:jupyter-mcp` -- MCP server implementation (sub-layer, 13 tools for programmatic notebook access)
- `/ov-jupyter:notebook-templates` -- Starter notebooks (data layer, used alongside this layer in images)
- `/ov-hermes:hermes` -- MCP consumer (auto-discovers via `OV_MCP_SERVERS` env var; uses `jupyter` server tools to read/edit/execute cells)
- `/ov-openwebui:openwebui` -- MCP consumer (sets `CODE_EXECUTION_ENGINE=jupyter` when this server is discovered, routing Open WebUI code-block execution into the Jupyter kernel)
- `/ov-foundation:supervisord` -- process manager dependency
- `/ov-foundation:python` -- Python runtime (transitive via supervisord)
- `/ov-build:mcp` -- end-to-end testing of the layer's MCP endpoint (`ov eval mcp ping`, `list-tools`, `call`); the layer ships 3 deploy-scope `mcp:` declarative checks against `list_notebooks`/`insert_cell`/`execute_cell`

## When to Use This Skill

Use when the user asks about:

- Real-time collaborative Jupyter notebooks
- jupyter-collaboration or Y-CRDT setup
- Lightweight Jupyter without GPU/ML dependencies
- Port 8888 service (check if GPU variant intended)
- The `jupyter` layer
- MCP server for notebook manipulation
- Programmatic notebook access from AI agents
- `watch_notebook` or change notifications
- Multi-client concurrent editing

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
