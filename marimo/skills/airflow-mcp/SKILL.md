---
name: airflow-mcp
description: |
  mcp-server-apache-airflow 0.2.10 wraps Apache Airflow's REST API as ~70 MCP tools (fetch_dags, post_dag_run, get_dag_run, list_connections, …).
  Use when working with the airflow MCP server's tool catalog, the JWT auth flow it shares with direct REST clients, or the wrapper script's airflow-readiness wait loop.
---

# airflow-mcp — Apache Airflow REST API exposed as MCP tools

`mcp-server-apache-airflow` is a third-party PyPI package
(`mcp-server-apache-airflow == 0.2.10`) that wraps Airflow's REST
API as MCP tools. Provides ~70 tools covering DAG management,
connections, variables, pools, datasets, task instances, and more —
essentially a 1:1 mirror of `/api/v2/` endpoints.

## URL pattern

```
http://{{.ContainerName}}:19999/mcp        # in-pod
http://localhost:29999/mcp                 # host-side (mapped port)
```

Transport: Streamable HTTP. Registered in this plugin's `.mcp.json`
as the `airflow` server.

## Wrapper script (`/usr/local/bin/airflow-mcp.sh`)

The wrapper script in `/ov-marimo:airflow-layer` does three things
before exec'ing the MCP server:

1. **Wait for api-server** — loops `curl -sf http://127.0.0.1:8080/api/v2/version`
   for up to 60 seconds. Without this, the MCP server's first
   forwarding call after a fresh boot races the api-server's startup
   and connection-refuses.
2. **Fetch JWT** — `POST /auth/token` with `admin` /
   `$AIRFLOW_ADMIN_PASSWORD`; passes the resulting `access_token`
   via `AIRFLOW_JWT_TOKEN` env (which `mcp-server-apache-airflow`
   prefers over `USERNAME` + `PASSWORD`).
3. **Fall back to basic auth** — if JWT fetch fails, sets
   `AIRFLOW_USERNAME` + `AIRFLOW_PASSWORD` so the package can retry as
   basic auth. (Airflow 3.x rejects basic on `/api/v2/` by default,
   so this fallback is best-effort — but at least the failure mode
   is visible in logs.)
4. **`exec`** — the MCP server with `--transport http --mcp-host
   0.0.0.0 --mcp-port 19999`. The `0.0.0.0` bind is mandatory so
   ov's host-port mapping reaches it from `127.0.0.1:HOST_PORT`.

## Headline tools (sample of ~70)

The full catalog mirrors `/api/v2/` 1:1. Headline tools you'll
typically use:

| Tool | Purpose |
|---|---|
| `fetch_dags` | List all DAGs registered with the scheduler |
| `get_dag` | Get a DAG by ID |
| `get_dag_details` | Simplified DAG representation |
| `get_dag_source` | Get the .py source code of a DAG |
| `pause_dag`, `unpause_dag` | Toggle DAG paused state |
| `patch_dag` | Update DAG attrs (e.g. unpause via `{"is_paused": false}`) |
| `post_dag_run` | **Trigger a DAG by ID** — the canonical write path |
| `get_dag_runs` | List runs for a DAG |
| `get_dag_run` | Get a specific run by `dag_run_id` |
| `update_dag_run_state` | Manually set a run state |
| `delete_dag_run` | Delete a run |
| `clear_dag_run` | Re-run all tasks in a DAG run |
| `get_dag_tasks`, `get_task`, `get_task_instance` | Task introspection |
| `clear_task_instances`, `set_task_instances_state` | Task admin |
| `get_log` | Fetch task log |
| `list_connections`, `create_connection`, `update_connection`, `delete_connection`, `test_connection` | Connection management |
| `list_variables`, `create_variable`, `update_variable`, `delete_variable` | Variable management |
| `get_pools`, `post_pool`, `patch_pool`, `delete_pool` | Pool management |
| `get_xcom_entries`, `get_xcom_entry` | XCom inspection |
| `get_event_logs`, `get_import_errors` | Diagnostics |
| `get_health`, `get_version` | Server health |

## Verification

Ping the server:

```bash
ov eval mcp ping marimo-ml-pod --name airflow
```

List tools (the eval suite already includes `mcp-airflow-list-tools`
as a deploy-scope probe):

```bash
ov eval mcp list-tools marimo-ml-pod --name airflow
```

Call a real RPC (list DAGs):

```bash
ov eval mcp call marimo-ml-pod fetch_dags '{"args":{}}' --name airflow
```

Trigger a DAG:

```bash
ov eval mcp call marimo-ml-pod post_dag_run '{"args":{
  "dag_id": "notebook_osm_pipeline",
  "conf": {},
  "logical_date": "2026-05-10T00:00:00Z"
}}' --name airflow
```

## Auth caveat

Airflow 3.x's auth is uneven across the mcp-server-apache-airflow
package's call sites:

- **JWT path (preferred)**: works for endpoints that the package
  passes `AIRFLOW_JWT_TOKEN` through to.
- **Basic auth fallback**: rejected by Airflow 3.x on most `/api/v2/`
  endpoints. The wrapper sets it anyway as a last-resort.

The eval suite deliberately does NOT enforce `mcp call` probes for
write operations — only `ping` and `list-tools` (which don't need
a working downstream). End-to-end DAG triggering is covered by the
direct REST API path used in `/ov-marimo:notebook-osm` (which
bypasses MCP entirely).

## MCP name decoupling

The MCP server name `airflow` is the service contract — declared in
`layers/airflow/layer.yml` `mcp_provides.name: airflow`. This plugin's
`.mcp.json` keys off the same name.

## Cross-references

- `/ov-marimo:airflow-layer` — layer that runs the server
- `/ov-marimo:marimo-mcp` — the OTHER MCP server in the same pod
- `/ov-marimo:notebook-osm` — uses direct REST instead of MCP for triggering
- `/ov-marimo:marimo-ml` — image composing the airflow layer
- `/ov-build:mcp` — MCP probe verb authoring + URL rewriter
- `/ov-eval:eval` — `ov eval mcp` subcommand reference
