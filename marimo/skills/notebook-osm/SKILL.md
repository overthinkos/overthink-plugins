---
name: notebook-osm
description: |
  Standalone marimo notebook (osm-monaco-viz.py) that self-authors TWO Airflow DAGs (osm + gtfs), triggers them via REST, runs polars + pyarrow analytics on both datasets, and renders TWO maps: streets via MapLibre GL JS + martin vector tiles + 3D terrain, transit via folium with 98 bus-stop CircleMarkers.
  Use when working with the notebook content itself, the dual-DAG self-authoring pattern, the two URL spaces (server-side AIRFLOW_API_INTERNAL_URL vs browser-bound MARTIN_PUBLIC_URL), the MapLibre/folium rendering split, or the surfaced-and-fixed bug catalog.
---

# notebook-osm — the Monaco OSM + GTFS demo notebook

A pure data layer that ships ONE file: a 10-cell self-contained
marimo notebook demonstrating the full marimo-ml stack end-to-end.

The notebook **self-authors its own Airflow DAGs at runtime** — no
DAG files are baked into the image. It writes both the OSM pipeline
DAG and the GTFS transit pipeline DAG into the airflow DAGs folder,
triggers them in parallel via the REST API, polls both to success,
then runs analysis + visualisation cells.

## Layer properties

| Property | Value |
|----------|-------|
| Type | Pure data layer (no packages, no services, no tasks) |
| Dependencies | none (file-only) |
| Data | `data/notebooks/` → `workspace:notebooks/` (deploy-time bind/seed) |
| Eval | `notebook-osm-viz-present` — file exists in volume |
| Notebook | `~/workspace/notebooks/osm-monaco-viz.py` (~10 cells) |

## Notebook architecture (10 cells, source order)

| # | Purpose | Returns |
|---|---|---|
| 1 | imports | mo, os, time, textwrap, requests, Path, polars, folium |
| 2 | markdown header (mo.md) | rendered intro + URL strategy table |
| 3 | self-author **both** DAG files | `dag_files`, `dag_ids = [osm, gtfs]`, `dags_dir` |
| 4 | trigger BOTH DAGs in parallel + poll until both succeed | `dag_run_states = {osm: success, gtfs: success}` |
| 5 | OSM GPU `geom_kb` histogram via cudf-polars-cu13 | `df_gpu`, `parquet_path` |
| 6 | OSM tag-key histogram via pyarrow (workaround) | `df_tags` |
| 7 | streets MapLibre GL JS map (vector tiles + 3D terrain) | renders inline iframe |
| 8 | GTFS analytics — stops/routes/trips counts + top routes | `df_route_stops`, `df_stops`, `gtfs_summary` |
| 9 | render `df_route_stops` as table | (display) |
| 10 | transit folium map (98 bus-stop CircleMarkers) | renders folium |

The two DAGs are independent (`notebook_osm_pipeline`,
`notebook_gtfs_pipeline`) but triggered + polled together by the
single trigger cell.

## DAG self-authoring pattern (cell 3)

The cell uses `textwrap.dedent('''...''')` to embed a complete DAG
file as a string literal, then `Path.write_text(...)` to drop it
into `${AIRFLOW_DAGS_DIR}` (defaults `~/workspace/dags`,
overridable for cross-pod). Idempotent — overwriting on every
notebook run keeps the DAG body in sync with the notebook (single
source of truth).

The OSM DAG ends with a `reload_martin` task that runs
`supervisorctl restart martin` after writing pmtiles — see
`/ov-marimo:osm-tools-layer` for why.

The GTFS DAG uses `gtfs_parquet.parse_gtfs(zip_path)` then
`write_parquet(dir)` to produce one `.parquet` per GTFS table.

## DAG trigger pattern (cell 4)

```python
# 1. Get JWT from /auth/token
# 2. For each DAG: poll /api/v2/dags/<id> until registered (max 90s,
#    given AIRFLOW__DAG_PROCESSOR__REFRESH_INTERVAL=10s)
# 3. Unpause if needed (PATCH /api/v2/dags/<id>)
# 4. Trigger (POST /api/v2/dags/<id>/dagRuns with logical_date)
# 5. Poll BOTH DAGs concurrently until each reaches success/failed
```

Returns a `dag_run_states` dict so dependent cells can gate on
`dag_run_states["notebook_osm_pipeline"] == "success"`.

## Two URL spaces (load-bearing design)

| Use | Where consumed | URL |
|---|---|---|
| Notebook → Airflow REST API | Inside the pod (Python `requests`) | `AIRFLOW_API_INTERNAL_URL` (default `http://localhost:8080`) |
| Folium / MapLibre tile URL | User's **browser** on the host | `MARTIN_PUBLIC_URL` (default `http://127.0.0.1:23000`) |

The marimo kernel runs INSIDE the pod (container-internal reach).
The user's browser runs OUTSIDE the pod (only host-port mappings
work). Mixing the two is the most common failure mode — see
`/ov-marimo:marimo-ml` "Required deploy.yml env block".

## Key gotchas (surfaced and fixed)

### 1. polars 1.38 cannot read quackosm `tags` column

The `tags` column is Arrow `Map<String,String>`. polars 1.38's
arrow2 MapArray reader decodes the logical type as
`List<List<Struct>>` and asserts a plain Struct inner — panics on
`.collect()`, `.read_parquet()`, AND `.collect(engine="gpu")`. NOT
GPU-specific.

**Workaround in cell 6**: read `tags` via `pyarrow.read_table` (which
handles `map<string,string>` correctly) + tally tag KEYS via
`collections.Counter`. The GPU cell (cell 5) avoids `tags` entirely
by grouping on a flat `geom_kb` column.

### 2. Folium "Make this Notebook Trusted" wrapper

Branca's `_repr_html_` (folium's HTML renderer) emits a
"trust" wrapper around the iframe `srcdoc` only when
`figure.height is None`. The wrapper hides the actual map content
in non-Jupyter renderers (marimo).

**Fix**: `m.get_root().height = "500px"` sets the **Figure's**
height (NOT the Map's) → branca takes the simple-iframe branch
without the trust wrapper. One-line fix using pure folium API.

### 3. Folium TileLayer is RASTER-only

Tippecanoe → pmtiles → martin produces VECTOR tiles
(Mapbox Vector Tile / PBF format,
`Content-Type: application/x-protobuf`). Folium's TileLayer treats
responses as PNG/JPG → silent failure → grey map.

**Fix in cell 7**: replaced folium with **MapLibre GL JS** via
`mo.iframe`. Vector source pointed at martin's TileJSON URL,
polygon/line/circle layers filtered by geometry-type.

### 4. Streets cell adds 3D terrain + hillshading

Per the official MapLibre 3D terrain example:

- `terrainSource` + `hillshadeSource` `raster-dem` sources at
  `https://tiles.mapterhorn.com/tilejson.json` (CORS-permissive)
- `hillshade` layer renders relief shading
- `terrain: { source: 'terrainSource', exaggeration: 1.5 }`
- `sky: {}` for atmospheric horizon
- `pitch: 60`, `maxPitch: 85`
- `TerrainControl` lets the user toggle terrain

The whole scene is elevated — the vector OSM data follows the
DEM topography.

### 5. Smart-download for both PBF and GTFS

Both `download_pbf` and `download_gtfs` HEAD-check `Last-Modified`
before re-fetching. Saves bandwidth on warm-cache re-runs (Geofabrik
~12 MB PBF + transitous.org ~770 KB GTFS).

## Verification (the Monaco bus network)

End-to-end run via marimo's own export (executes every cell
server-side):

```bash
podman exec ov-marimo-ml-pod /home/user/.pixi/envs/default/bin/marimo \
  export ipynb /home/user/workspace/notebooks/osm-monaco-viz.py \
  --include-outputs --sort topological -o /tmp/notebook-run.ipynb -f
```

Expected results (verified):

- Cell 4: `{"notebook_osm_pipeline": "success", "notebook_gtfs_pipeline": "success"}`
- Cell 5: polars DataFrame shape (9, 2) — geom_kb histogram
- Cell 6: polars DataFrame shape (20, 2), top key `'highway'` count 4571
- Cell 7: MapLibre HTML embeds martin URL + mapterhorn DEM, no trust wrapper
- Cell 8: GTFS counts: 98 stops, 15 routes, 5846 trips, 80581 stop_times
- Cell 9: top route `'5'` with 46 distinct stops
- Cell 10: folium HTML contains 98 `circleMarker` calls

## Cross-references

- `/ov-marimo:marimo-ml` — image composing this layer
- `/ov-marimo:marimo-layer` — the runtime kernel
- `/ov-marimo:airflow-layer` — DAG trigger REST API
- `/ov-marimo:osm-tools-layer` — martin + tippecanoe + reload pattern
- `/ov-marimo:maputnik-layer` — companion vector-tile editor
- `/ov-marimo:marimo-mcp` — read-only inspection of this notebook's session
