---
name: marimo-ml
description: |
  marimo reactive notebook environment with Apache Airflow + GPU-accelerated OSM/GTFS analytics + martin vector tiles + 3D terrain via MapLibre.
  Composes 9 layers (agent-forwarding, marimo, airflow, osm-data, maputnik, notebook-osm, debug-tools, dbus, ov) into a single pod that exposes 5 host ports and 2 MCP servers.
  MUST be invoked before building, deploying, configuring, or troubleshooting the marimo-ml image.
---

# marimo-ml — marimo + Airflow + OSM/GTFS + martin

GPU-accelerated marimo notebook environment that composes a full
analytics + visualisation stack: marimo notebooks (also acting as MCP
server), Apache Airflow (LocalExecutor + SQLite), OSM data pipeline
(quackosm → tippecanoe → martin), GTFS transit pipeline (gtfs-parquet),
and a MapLibre style editor (maputnik). Two MCP servers are exposed
(marimo for notebook diagnostics, airflow for REST-API orchestration).

## Image properties

| Property | Value |
|----------|-------|
| Base | `nvidia` (CUDA 13 / cuDNN / ONNX Runtime on Fedora 43 nonfree) |
| Platforms | `linux/amd64` only (cuDF + cu130 torch are amd64-only) |
| Layers | 9 (see "Layer stack" below) |
| Ports | 5 (see "Ports + host mappings") |
| MCP servers | 2 (marimo @ container 2718, airflow @ container 19999) |
| Registry | `ghcr.io/overthinkos` |
| Image tag pattern | CalVer (`YYYY.DDD.HHMM`) |

## Layer stack (composition order)

1. `nvidia` (base — CUDA 13 / cuDNN / ONNX Runtime)
2. `agent-forwarding` — SSH/GPG agent forwarding for in-container git push
3. `marimo` — `/ov-marimo:marimo-layer` — pixi env (cudf-polars-cu13,
   torch cu130, geopandas, quackosm, gtfs-parquet, folium, marimo,
   airflow Python deps), supervisord service `marimo edit … --mcp`
4. `airflow` — `/ov-marimo:airflow-layer` — 4 supervisord services
   (init/scheduler/dag-processor/webserver) + the airflow-mcp wrapper
5. `osm-data` — `/ov-marimo:osm-data-layer` — tippecanoe (built from
   source), osmium, gdal, jq, martin (Rust), pmtiles CLI
6. `maputnik` — `/ov-marimo:maputnik-layer` — MapLibre style editor
   built with Vite `--base=/`
7. `notebook-osm` — `/ov-marimo:notebook-osm` — data-only layer
   seeding the dual-DAG OSM+GTFS notebook into `~/workspace/notebooks/`
8. `debug-tools` — `/ov-marimo:debug-tools-layer` — 49 standard
   debug utilities (network/process/file/system/session)
9. `dbus` — D-Bus session bus
10. `ov` — `ov` CLI binary inside the container

## Ports + host mappings

The image declares container ports; `marimo-ml-pod` deploy entry
remaps to host ports for browser reachability:

| Service | Container port | Host port | Path |
|---|---|---|---|
| marimo edit + MCP | 2718 | **22718** | `/` for editor, `/mcp/server` for MCP |
| airflow api-server | 8080 | **28080** | `/api/v2/`, `/auth/token`, `/api/v2/dags/<id>/dagRuns` |
| airflow MCP | 19999 | **29999** | `/mcp` |
| martin tile server | 3000 | **23000** | `/<source>/{z}/{x}/{y}` (vector tiles), `/<source>` (TileJSON), `/catalog` |
| maputnik static editor | 8000 | **28000** | `/` (SPA root) |

**Host port 8080 is NOT used** for airflow because traefik already
binds it on the operator's machine. Always use **28080** for airflow
from the host.

## Required deploy.yml env block

The user's browser is on the host; container-internal `localhost:N`
URLs do NOT resolve there. The notebook reads four env vars to
bridge the two URL spaces:

```yaml
marimo-ml-pod:
  target: pod
  image: marimo-ml
  disposable: true
  lifecycle: dev
  ports: [22718:2718, 28080:8080, 29999:19999, 23000:3000, 28000:8000]
  env:
    # Browser-bound (embedded in folium/MapLibre HTML)
    - "MARTIN_PUBLIC_URL=http://127.0.0.1:23000"
    - "AIRFLOW_PUBLIC_URL=http://127.0.0.1:28080"
    # Notebook-internal (server-side requests in the kernel)
    - "AIRFLOW_API_INTERNAL_URL=http://localhost:8080"
    - "AIRFLOW_DAGS_DIR=/home/user/workspace/dags"
```

For cross-pod topologies (airflow in a separate pod on the shared
`ov` podman network), override `AIRFLOW_API_INTERNAL_URL` to
`http://airflow-pod:8080` and ensure both pods mount the same
`workspace` volume at the same path.

## MCP servers

This image publishes two MCP endpoints (registered by this plugin's
`.mcp.json`):

| Name | Container URL | Tool count | Skill |
|---|---|---:|---|
| marimo | `http://localhost:2718/mcp/server` | 10 (read-only inspection) | `/ov-marimo:marimo-mcp` |
| airflow | `http://localhost:19999/mcp` | ~70 (REST API wrappers) | `/ov-marimo:airflow-mcp` |

## R10 acceptance

```bash
ov update --build --force-seed marimo-ml-pod
ov eval live marimo-ml-pod         # → 83 passed · 0 failed · 0 skipped
```

End-to-end notebook test (executes all 10 cells via marimo's own
export):

```bash
podman exec ov-marimo-ml-pod /home/user/.pixi/envs/default/bin/marimo \
  export ipynb /home/user/workspace/notebooks/osm-monaco-viz.py \
  --include-outputs --sort topological -o /tmp/notebook-run.ipynb -f
```

Expected outputs (verified end-to-end):

- both DAGs reach `state: success`
- OSM parquet ~824 KB at `~/workspace/tiles/work/monaco.parquet`
- pmtiles ~2.6 MB at `~/workspace/tiles/pmtiles/monaco.pmtiles`
- GTFS zip ~770 KB at `~/workspace/gtfs/raw/monaco.gtfs.zip`
- 98 stops, 15 routes, 5846 trips in `~/workspace/gtfs/parquet/`
- streets MapLibre HTML embeds martin URL, mapterhorn DEM, hillshade,
  terrain, sky, TerrainControl
- transit folium HTML embeds 98 `circleMarker` calls
- neither folium map carries the "Make this Notebook Trusted" wrapper

## Known gotchas

- **User deploy.yml needs `image:` field.** `~/.config/ov/deploy.yml`
  marimo-ml-pod entry must include `image: marimo-ml` — otherwise
  `ov update --build` fails with "deploy has no 'image:' field".
- **Marimo session persistence reorders cells.** When a marimo session
  is open and the file changes (e.g. after `--force-seed`), marimo
  re-persists with its session's cell order. Surgical reset:
  `supervisorctl stop marimo && cp /data/workspace/notebooks/osm-monaco-viz.py /home/user/workspace/notebooks/ && supervisorctl start marimo`.
- **Martin caches pmtiles mtime at startup.** Re-run a DAG that
  rewrites `monaco.pmtiles` → martin returns 500/204 forever until
  restart. The OSM DAG's `reload_martin` task handles this automatically;
  see `/ov-marimo:osm-data-layer`.
- **Notebook needs `image:` field too.** Already covered above —
  same point worth emphasizing.

## Cross-references

- `/ov-marimo:marimo-layer` — marimo runtime layer (pixi env, service)
- `/ov-marimo:marimo-mcp` — marimo MCP server tool catalog
- `/ov-marimo:airflow-layer` — Airflow 3.x compat findings
- `/ov-marimo:airflow-mcp` — airflow MCP server tool catalog
- `/ov-marimo:notebook-osm` — the dual-DAG OSM+GTFS notebook
- `/ov-marimo:maputnik-layer` — Vite `--base=/` build pattern
- `/ov-marimo:osm-data-layer` — martin reload pattern
- `/ov-marimo:debug-tools-layer` — debug toolkit
- `/ov-build:eval` — eval verbs used by the 83 probes
- `/ov-core:deploy` — env block authoring + cross-pod topology
- `/ov-dev:disposable` — `disposable: true` semantics
