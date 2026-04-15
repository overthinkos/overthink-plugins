# ov-jupyter

MCP server registration for JupyterLab notebook operations with real-time CRDT collaboration.

This plugin exposes 13 MCP tools (`list_notebooks`, `get_notebook`, `create_notebook`, `get_cell`, `update_cell`, `insert_cell`, `delete_cell`, `execute_cell`, `open_notebook_session`, `close_notebook_session`, `watch_notebook`, `get_active_users`, `get_active_sessions`) to Claude Code, routing them to the JupyterLab server running on `http://localhost:8888/mcp`.

## Contents

- `.claude-plugin/plugin.json` — plugin metadata (name, description, version)
- `.mcp.json` — MCP server registration pointing at `http://localhost:8888/mcp`

No skills of its own — the backing documentation lives in `/ov-layers:jupyter-mcp` (server implementation) and the three image skills that publish the server.

## Requirements

- A `jupyter` / `jupyter-ml` / `jupyter-ml-notebook` container must be **running** before Claude Code starts. See `/ov-images:jupyter`, `/ov-images:jupyter-ml`, `/ov-images:jupyter-ml-notebook`.
- Claude Code registers MCP servers at **session start**. If the Jupyter container is launched *after* Claude Code, the auto-connect to `http://localhost:8888/mcp` fails silently and the `mcp__jupyter__*` tools will not appear in the session's tool catalog. **Restart Claude Code after the container is up** to pick up the registration. There is no runtime reconnect path today.

## MCP Name Decoupling

The MCP server name `jupyter` is deliberately stable across renames of the underlying layer, Python package, and image. The `/ov-layers:jupyter-mcp` skill documents this pattern in detail ("MCP Name Decoupling" section). Short version: `mcp_provides.name: jupyter` in `layers/jupyter/layer.yml` is the service contract — consumers (`hermes`, `openwebui`, this plugin's `.mcp.json`) all key off that name, not off the layer/package/image name.

## Related Skills

- `/ov-layers:jupyter-mcp` — the MCP server implementation (13 tools, FastMCP + Tornado-ASGI bridge, CRDT integration, design principles)
- `/ov-layers:jupyter` — lightweight parent layer (composes `jupyter-mcp`)
- `/ov-layers:jupyter-ml` — GPU ML parent layer (composes `llama-cpp`, `unsloth`, `jupyter-mcp`)
- `/ov-images:jupyter` — lightweight CPU image
- `/ov-images:jupyter-ml` — GPU image
- `/ov-images:jupyter-ml-notebook` — GPU image + seeded finetuning/ollama/openrouter/llms_on_supercomputers notebooks
- `/ov-layers:hermes` — another MCP consumer that discovers the `jupyter` server via cross-container `env_provides` / `mcp_provides`
