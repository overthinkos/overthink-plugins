---
name: airflow-layer
description: |
  Apache Airflow 3.x with LocalExecutor + SQLite (single-node, dev-friendly), 4 supervisord services (init, scheduler, dag-processor, webserver) plus the airflow-mcp wrapper. Layer is service-only — its Python deps live in /ov-marimo:marimo-layer's pixi env.
  Use when working with the airflow layer, Airflow 3.x compatibility findings, the SimpleAuthManager auth-fix pattern, the dag-processor split-from-scheduler architecture change, or the JWT-issuance + REST API trigger flow used by self-authoring notebooks.
---

# airflow — Apache Airflow 3.x layer

LocalExecutor + SQLite — zero external services. Suitable for
single-node dev / R10 verification. For multi-node production swap
in CeleryExecutor + Postgres + Valkey (those layers stayed available
under `layers/postgresql/` + `layers/valkey/`).

This layer is **service-only**: it ships no `pixi.toml`. Its Python
deps (`apache-airflow`, `mcp-server-apache-airflow`, `fastmcp`) live
in `/ov-marimo:marimo-layer`'s pixi env, so airflow runs alongside
marimo in the same pod with one combined Python environment.

## Layer properties

| Property | Value |
|----------|-------|
| Dependencies | `marimo` (for the pixi env), `supervisord` |
| Ports | `8080` (api-server, host-mapped to **28080**), `19999` (airflow-mcp HTTP) |
| Services | `airflow-init` (one-shot) + `airflow-scheduler` + `airflow-dag-processor` + `airflow-webserver` + `airflow-mcp` |
| Volumes | `airflow-data` at `~/airflow` |
| Secrets | `airflow-fernet-key`, `airflow-webserver-secret`, `airflow-admin-password` |
| MCP provides | `airflow` at `http://{{.ContainerName}}:19999/mcp` (Streamable HTTP) |
| DAG folder | `~/workspace/dags` (`AIRFLOW__CORE__DAGS_FOLDER`) |

## Airflow 3.x compatibility — 8 RCA findings

The layer was hardened across multiple RCA cycles to handle Airflow
3.x's architectural changes. Each finding is encoded in either the
init script, env vars, or supervisord service list:

### 1. SimpleAuthManager replaced FAB

Airflow 3.x removed `airflow users create`. The new SimpleAuthManager
generates a **random** admin password on first boot and writes it to
`${AIRFLOW_HOME}/simple_auth_manager_passwords.json.generated` —
that random value bears no relation to the `AIRFLOW_ADMIN_PASSWORD`
secret we inject, so JWT auth via `/auth/token` rejects every request
with 401.

**Fix in `airflow-init.sh`**: overwrite the passwords file from
`${AIRFLOW_ADMIN_PASSWORD}` on every boot. Idempotent; tolerant of
secret rotation.

```bash
mkdir -p "${HOME}/airflow"
printf '{"admin": "%s"}\n' "${AIRFLOW_ADMIN_PASSWORD}" \
  > "${HOME}/airflow/simple_auth_manager_passwords.json.generated"
chmod 0600 "${HOME}/airflow/simple_auth_manager_passwords.json.generated"
```

### 2. dag_dir_list_interval moved to dag_processor section

The Airflow-2 key path `[scheduler] dag_dir_list_interval` is
deprecated in 3.x; the canonical key is now
`[dag_processor] refresh_interval`. Layer env block sets:

```yaml
env:
  AIRFLOW__DAG_PROCESSOR__REFRESH_INTERVAL: "10"
```

Default is 300 seconds — far too slow for the runtime-DAG-write
pattern (notebook drops a `.py` and triggers it within seconds). 10s
makes the scan effectively interactive at negligible CPU cost on a
single-user dev pod.

### 3. DAG processor split from scheduler

Airflow 3.x splits the DAG processor out of the scheduler — without a
separate `airflow dag-processor` daemon, the dags folder is **never
scanned** (notebooks waiting on registration deadlock forever).

**Fix**: new `airflow-dag-processor` supervisord service alongside
`airflow-scheduler`. Without this service, `dag_processor.refresh_interval`
has no effect because nothing honors it.

### 4. POST /dagRuns requires logical_date

Airflow 2.x auto-generated `logical_date` if omitted; 3.x rejects with
422. Notebook trigger code must include:

```python
from datetime import datetime, timezone
requests.post(f"{api}/api/v2/dags/{dag_id}/dagRuns",
              headers=auth,
              json={"conf": {}, "logical_date": datetime.now(timezone.utc).isoformat()},
              timeout=10)
```

### 5. webserver renamed to api-server

Airflow 3.x dropped `airflow webserver` in favor of `airflow api-server`
(FastAPI-based). The `airflow-webserver.sh` wrapper was updated:

```bash
exec "${HOME}/.pixi/envs/default/bin/airflow" api-server
```

### 6. Auth backend defaults to JWT

The api-server uses JWT issued by SimpleAuthManager. Clients fetch
a token via `POST /auth/token` (admin/password JSON body), then pass
`Authorization: Bearer <token>` on subsequent calls. Basic auth is
rejected on `/api/v2/` endpoints by default.

### 7. mcp-server-apache-airflow needs `--mcp-host`

The package (0.2.10) defaults `--mcp-host` to `127.0.0.1`. We bind
`0.0.0.0` so the host-port mapping reaches it:

```bash
exec "${HOME}/.pixi/envs/default/bin/mcp-server-apache-airflow" \
  --transport http --mcp-host 0.0.0.0 --mcp-port 19999
```

### 8. airflow-init must NOT block boot

`airflow-init` is a one-shot supervisord program (`restart: "no"`,
`exit_codes: "0"`) that runs `airflow db migrate` + writes the
passwords file. It exits cleanly on success; the long-running daemons
start at higher priority numbers (30+) once init has finished.

## Service list (5 entries)

| Name | Priority | Restart | Purpose |
|---|---:|---|---|
| `airflow-init` | 25 | no | `airflow db migrate` + write SimpleAuthManager passwords file |
| `airflow-scheduler` | 30 | always | Schedule DAG runs from the metadata DB |
| `airflow-dag-processor` | 31 | always | Scan dags folder + serialise DAGs into the DB |
| `airflow-webserver` | 31 | always | `airflow api-server` (FastAPI on :8080) |
| `airflow-mcp` | 32 | always | mcp-server-apache-airflow wrapper on :19999 |

## Eval probes (12 — proves the auth + REST + MCP paths work)

Build-scope (5):
- `airflow-init-script` `/usr/local/bin/airflow-init.sh` exists + executable
- `airflow-webserver-script` ditto
- `airflow-scheduler-script` ditto
- `airflow-dag-processor-script` ditto (added with #3 above)
- `airflow-mcp-script` ditto + `airflow-mcp-binary` (the pixi-installed binary)

Deploy-scope (7):
- `airflow-init-db-created` — `~/airflow/airflow.db` exists (SQLite metadata)
- `airflow-scheduler-running`, `airflow-dag-processor-running`, `airflow-webserver-running`, `airflow-mcp-running` — supervisord program states
- `airflow-port-reachable` — TCP 8080 reachable
- `airflow-http-version` — `GET /api/v2/version` returns 200
- `airflow-jwt-issuance` — `POST /auth/token` with admin creds returns body containing `access_token` (THE notebook auth path; locks in finding #1)
- `airflow-mcp-port-reachable` — TCP 19999 reachable
- `mcp-airflow-ping` + `mcp-airflow-list-tools` — MCP server alive + exposes `fetch_dags`, `post_dag_run`, `get_dag_run`

## Notebook self-author DAG pattern

The marimo notebook in `/ov-marimo:notebook-osm` writes its own DAG
file into `~/workspace/dags/` and triggers it via the REST API:

1. POST `/auth/token` (admin / `$AIRFLOW_ADMIN_PASSWORD`) → JWT
2. Wait for the DAG to register: poll `GET /api/v2/dags/<id>`
   (typically <30 s with `REFRESH_INTERVAL=10`)
3. Unpause: PATCH `/api/v2/dags/<id>` with `{"is_paused": false}`
4. Trigger: POST `/api/v2/dags/<id>/dagRuns` with `logical_date`
5. Poll: GET `/api/v2/dags/<id>/dagRuns/<run_id>` until `state in
   ("success", "failed")`

The notebook uses this pattern to fire BOTH `notebook_osm_pipeline`
and `notebook_gtfs_pipeline` in parallel.

## Cross-references

- `/ov-marimo:airflow-mcp` — the airflow-mcp tool catalog
- `/ov-marimo:marimo-layer` — pixi env that owns the airflow Python deps
- `/ov-marimo:notebook-osm` — canonical user of the REST trigger pattern
- `/ov-marimo:marimo` — the image composing this layer
- `/ov-infrastructure:supervisord` — service mgmt
- `/ov-build:secrets` — the 3 airflow secrets injected via this layer
