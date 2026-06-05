---
name: marimo-mcp
description: |
  marimo's built-in MCP server (10 read-only inspection tools — get_active_notebooks, get_cell_outputs, get_notebook_errors, etc.) at port 2718 path /mcp/server.
  Use when working with the marimo MCP tool catalog, programmatic notebook diagnostics, or the cells-don't-execute-via-MCP gap (marimo MCP is read-only; cells run via WebSocket from a browser OR `marimo export ipynb --include-outputs` for headless execution).
---

# marimo-mcp — marimo's built-in MCP server (read-only inspection)

marimo's notebook server has a built-in MCP endpoint enabled by the
`--mcp` flag (see `/ov-versa:versa-layer` service spec). It serves
10 inspection tools for diagnosing active notebook sessions —
**read-only**: cells cannot be executed via this MCP. Execution
requires a browser-attached WebSocket session OR
`marimo export ipynb --include-outputs` for headless runs.

## URL pattern

```
http://{{.ContainerName}}:2718/mcp/server   # in-pod
http://localhost:22718/mcp/server           # host-side (mapped port)
```

Transport: Streamable HTTP. Registered in this plugin's `.mcp.json`
as the `marimo` server.

## Tool catalog (10 tools)

All tools are **read-only**. Each takes an `args` JSON object as
the sole parameter (often `{}` or `{"session_id": "..."}`).

| Tool | Purpose |
|---|---|
| `get_marimo_rules` | Returns the official marimo rules for AI assistants |
| `get_active_notebooks` | Lists currently-open notebooks + session IDs + active connection counts |
| `get_lightweight_cell_map` | Per-cell preview (cell_id, line_count, runtime_state, has_output, has_errors) for a session |
| `get_cell_runtime_data` | Full runtime data for one or more cells: code, errors, declared variables |
| `get_cell_outputs` | Cell execution outputs (visual display + console streams) |
| `get_tables_and_variables` | All tables and variables visible in a session |
| `get_database_tables` | Database table info (regex query supported) |
| `get_notebook_errors` | All errors in a session, organised by cell |
| `lint_notebook` | Lint a marimo notebook for issues |
| `get_cell_dependency_graph` | Cell dependency graph (variable-flow edges) |

## Why no execute tool

marimo's reactive runtime executes cells based on dependency-graph
analysis when variable values change in the editor. The MCP exposes
a *read* surface against an active session — there's no "run cell N"
RPC because that's not how marimo orchestrates cells.

Two paths to execute notebook content programmatically:

1. **Browser-attached run** — open the notebook in a browser; marimo
   reactively runs all reachable cells. The marimo MCP can then read
   the session state (cell outputs, errors, variables).

2. **Headless export** — `marimo export ipynb --include-outputs` runs
   every cell server-side and writes a Jupyter notebook with embedded
   outputs. Used in this repo's R10 acceptance for the
   osm-monaco-viz.py notebook:

   ```bash
   podman exec ov-versa /home/user/.pixi/envs/default/bin/marimo \
     export ipynb /workspace/notebooks/osm-monaco-viz.py \
     --include-outputs --sort topological -o /tmp/notebook-run.ipynb -f
   ```

   Requires `nbformat` in the pixi env (already pinned in
   `candy/marimo/pixi.toml`).

## Verification

Ping the server (proves it's alive + reachable):

```bash
ov eval mcp ping marimo --name marimo
```

List the tools (catalog enumeration):

```bash
ov eval mcp list-tools marimo --name marimo
```

Inspect a session's cell map (real diagnostic):

```bash
SID=$(ov eval mcp call versa get_active_notebooks '{"args":{}}' --name marimo \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["data"]["notebooks"][0]["session_id"])')
ov eval mcp call versa get_lightweight_cell_map "{\"args\":{\"session_id\":\"$SID\"}}" --name marimo
```

## MCP name decoupling

The MCP server name `marimo` is the service contract — declared in
`candy/marimo/candy.yml` `mcp_provide.name: marimo`. This plugin's
`.mcp.json` keys off the same name. Renames of the layer / Python
package / image MUST NOT change this name unless the contract is
explicitly broken in a hard cutover.

## Cross-references

- `/ov-versa:versa-layer` — layer that runs the server
- `/ov-versa:airflow-mcp` — the OTHER MCP server in the same pod
- `/ov-versa:notebook-osm` — example notebook diagnosed via this MCP
- `/ov-build:ov-mcp-cmd` — MCP probe verb authoring + URL rewriter
- `/ov-eval:eval` — `ov eval mcp` subcommand reference
