---
name: versa
description: |
  versa — image bundling marimo reactive notebook environment with Apache Airflow + GPU-accelerated OSM/GTFS analytics + martin vector tiles + 3D terrain via MapLibre + Polars geospatial extensions (polars-st, geopolars) + GeoArrow + deck.gl rendering via lonboard.
  Composes 11 layers (agent-forwarding, nvidia, cuda, marimo, airflow, osm-tools, maputnik, notebook-osm, debug-tools, dbus, ov) on a CachyOS / Arch base into a single pod that exposes 5 host ports and 2 MCP servers.
  MUST be invoked before building, deploying, configuring, or troubleshooting the versa image.
---

# versa — marimo + Airflow + OSM/GTFS + martin

GPU-accelerated marimo notebook environment that composes a full
analytics + visualisation stack: marimo notebooks (also acting as MCP
server), Apache Airflow (LocalExecutor + SQLite), OSM data pipeline
(quackosm → tippecanoe → martin), GTFS transit pipeline (gtfs-parquet),
and a MapLibre style editor (maputnik). Two MCP servers are exposed
(marimo for notebook diagnostics, airflow for REST-API orchestration).

The image was renamed from `marimo` to `versa` in 2026-05 to decouple
OUR image identity from the upstream marimo notebook software the
image bundles. The `layers/marimo/` directory + the
`mcp_provide.name: marimo` MCP-server identity stayed unchanged.

## Image properties

| Property | Value |
|----------|-------|
| Base | `cachyos` (CUDA 13 / cuDNN / python-onnxruntime-cpu via CachyOS `extra` repo) |
| Platforms | `linux/amd64` only (cuDF + cu130 torch are amd64-only) |
| Layers | 18 (see "Layer stack" below) |
| Ports | 7 (see "Ports + host mappings") |
| MCP servers | 1 (marimo @ container 2718) — the upstream airflow MCP wrapper was removed in 2026-05 (no Airflow-3 / `/api/v2` release exists) |
| Registry | `ghcr.io/overthinkos` |
| Image tag pattern | CalVer (`YYYY.DDD.HHMM`) |
| Builder | `archlinux-builder` (pixi/npm/cargo/aur) — overrides the fedora-builder default since cachyos has no `builder:` of its own |

## Layer stack (composition order)

1. `cachyos` (base — `docker.io/cachyos/cachyos-v3:latest` OCI image, matches the upstream CachyOS docker repo workflow)
2. `agent-forwarding` — SSH/GPG agent forwarding for in-container git push
3. `nvidia` — driver runtime (`nvidia-utils`, `nvidia-container-toolkit`)
   — formerly inherited via `base: nvidia` (Fedora image), now an
   explicit layer entry since the cachyos base doesn't bundle it
4. `cuda` — CUDA toolkit (`cuda` + `cudnn` + `python-onnxruntime-cpu`
   from CachyOS extra repo; `/opt/cuda` symlinked into
   `/usr/{bin,include,lib64}` for path compatibility with the Fedora
   layout). No AUR builds remain in this image after the osm-tools
   rename — `build:` is `[pac]` only.
5. `marimo` — `/ov-versa:versa-layer` — pixi env (cudf-polars-cu13,
   torch cu130, geopandas, quackosm, gtfs-parquet, folium, marimo,
   airflow Python deps, plus polars-st / geopolars / geoarrow-pyarrow /
   geoarrow-pandas / lonboard for Polars-native spatial ops + deck.gl
   rendering, plus freestiler for direct DuckDB → PMTiles tile
   generation — gpq-tiles is system-installed via the osm-tools layer
   instead, because its PyPI wheels don't cover Python 3.13/Linux x86_64
   and the pixi env's `no-build = true` blocks sdist resolution),
   supervisord service `marimo edit … --mcp`
6. `airflow` — `/ov-versa:airflow-layer` — 4 supervisord services
   (init/scheduler/dag-processor/webserver). NB: airflow-mcp wrapper
   removed 2026-05 — no Airflow-3 v2 release of
   `mcp-server-apache-airflow` exists; consumers drive Airflow via
   direct REST `/api/v2/` calls.
7. `osm-tools` — `/ov-versa:osm-tools-layer` — tippecanoe (built from
   source), gdal, jq, martin (Rust binary), pmtiles CLI (Go binary),
   gpq-tiles (cargo-installed Rust binary; v0.6.0 from crates.io —
   the system-install fallback for the PyPI-wheel gap)
8. `maputnik` — `/ov-versa:maputnik-layer` — MapLibre style editor
   built with Vite `--base=/`
9. `pmtiles-viewer` — `/ov-versa:pmtiles-viewer` — protomaps/PMTiles
   /app SPA, visual inspector for PMTiles archives (port 8001 → 28001)
10. `shortbread` — `/ov-versa:shortbread` — systemed/tilemaker (C++/Lua)
    + shortbread-tilemaker config — produces shortbread-schema vector
    tiles into `/workspace/tiles/shortbread/`
11. `versatiles-style` — `/ov-versa:versatiles-style` — @versatiles/style
    MapLibre style generator bundled to `/opt/versatiles-style/`
12. `versatiles-fonts` — `/ov-versa:versatiles-fonts` — SDF font glyphs
    (Noto Sans + 9 others) bundled to `/opt/versatiles-fonts/`
13. `maplibre-versatiles-styler` — `/ov-versa:maplibre-versatiles-styler` —
    interactive MapLibre style-switcher control bundled to
    `/opt/maplibre-versatiles-styler/`
14. `versatiles` — `/ov-versa:versatiles` — versatiles-rs CLI binary
    (convert/serve/probe) + `versatiles serve` supervisord service on
    port 8090 → 28090 (parallel tile server to martin)
15. `versatiles-frontend` — `/ov-versa:versatiles-frontend` —
    versatiles-org/versatiles-frontend SPA (port 8002 → 28002); also
    re-exports versatiles-style / fonts / styler at `/style/`,
    `/fonts/`, `/styler/`
16. `notebook-osm` — `/ov-versa:notebook-osm` — data-only layer
    seeding the OSM+GTFS+pipelines notebook into `/workspace/notebooks/`
17. `debug-tools` — `/ov-versa:debug-tools-layer` — 49 standard
    debug utilities (network/process/file/system/session)
18. `dbus` — D-Bus session bus
19. `ov` — `ov` CLI binary inside the container

## Ports + host mappings

The image declares container ports; the `versa` deploy entry
remaps to host ports for browser reachability:

| Service | Container port | Host port | Path |
|---|---|---|---|
| marimo edit + MCP | 2718 | **22718** | `/` for editor, `/mcp/server` for MCP |
| airflow api-server | 8080 | **28080** | `/api/v2/`, `/auth/token`, `/api/v2/dags/<id>/dagRuns` |
| martin tile server | 3000 | **23000** | `/<source>/{z}/{x}/{y}` (vector tiles), `/<source>` (TileJSON), `/catalog` |
| maputnik static editor | 8000 | **28000** | `/` (SPA root) |
| pmtiles-viewer SPA | 8001 | **28001** | `/` (SPA root — load remote PMTiles archive via the UI's URL input) |
| versatiles-frontend SPA | 8002 | **28002** | `/` (SPA root); `/style/` re-exports versatiles-style bundle |
| versatiles serve | 8090 | **28090** | `/tiles/<source>/{z}/{x}/{y}.pbf` (parallel tile server to martin; serves /workspace/tiles/shortbread/) |

**Host port 8080 is NOT used** for airflow because traefik already
binds it on the operator's machine. Always use **28080** for airflow
from the host.

## Required deploy.yml entry

Starting with the 2026-05 port + env automation cutover, the versa
deploy entry is minimal — the seven URL env vars the notebook reads
are all auto-injected via per-producing-layer `env_provides:` blocks,
and the eight container ports are auto-allocated to free host ports
when the operator writes `port: [auto]`. The user's browser is on the
host; container-internal `localhost:N` URLs do NOT resolve there —
but the auto-derived `*_PUBLIC_URL` vars correctly carry the
host-side mapping, so the browser-bound and kernel-bound URL spaces
stay in sync automatically:

```yaml
versa:
  target: pod
  image: versa
  disposable: true
  lifecycle: dev
  port: [auto]    # auto-allocate one free host port per image-declared
                  # container port. The resolved expansion is persisted
                  # as `resolved_port:` alongside this entry on the
                  # next `ov config versa` / `ov update versa` run.
```

That's the entire entry. No `env:` block — the seven URL env vars
flow from layer `env_provides:`:

| env var | Producer layer | Template |
|---|---|---|
| `MARTIN_PUBLIC_URL` | `osm-tools` | `http://127.0.0.1:{{.HostPort 3000}}` |
| `AIRFLOW_PUBLIC_URL` | `airflow` | `http://127.0.0.1:{{.HostPort 8080}}` |
| `AIRFLOW_API_INTERNAL_URL` | `airflow` | `http://{{.ContainerName}}:8080` (rewritten to `localhost:8080` same-pod) |
| `AIRFLOW_DAGS_DIR` | `airflow` | `/workspace/dags` (static) |
| `VERSATILES_PUBLIC_URL` | `versatiles` | `http://127.0.0.1:{{.HostPort 8090}}` |
| `VERSATILES_STYLE_PUBLIC_URL` | `versatiles-frontend` | `http://127.0.0.1:{{.HostPort 8002}}/style` |
| `VERSATILES_ASSETS_PUBLIC_URL` | `versatiles-frontend` | `http://127.0.0.1:{{.HostPort 8002}}` |

If you need stable host ports across rebuilds (e.g. browser
bookmarks), replace `port: [auto]` with an explicit list — the
`env_provides:` templates substitute against whichever ports you
chose, so the URL env vars stay correct either way:

```yaml
port:
  - "22718:2718"
  - "28080:8080"
  - "23000:3000"
  - "28000:8000"
  - "28001:8001"
  - "28002:8002"
  - "28090:8090"
```

For cross-pod topologies (airflow in a separate pod on the shared
`ov` podman network), no special handling needed — the
`AIRFLOW_API_INTERNAL_URL` template renders to
`http://<airflow-pod-name>:8080` because `{{.ContainerName}}`
resolves to the airflow image's container, and `podAwareEnvProvides`
only rewrites to `localhost` when consumer and producer share a pod.

**To verify the resolved URLs once the deploy is running**, open the
notebook — the new diagnostic cell at the top (`_resolved_urls`)
renders a polars DataFrame listing every URL env var, its current
value, and whether it came from `env_provides` injection or the
fallback default. See `/ov-versa:notebook-osm` cell #2.

### Multi-instance pattern (Pattern A from /ov-core:deploy)

Run multiple instances of versa side-by-side using the
`<base>/<instance>` deploy-key form. Each instance gets its own
container (`ov-versa`, `ov-versa-ecovoyage`, …), its own workspace
volume, and its own host-port mappings:

```yaml
deploy:
  versa:
    image: versa
    target: pod
    disposable: true
    port: [auto]               # auto-allocated host ports

  versa/ecovoyage:
    image: versa               # SAME image, explicit field required
    target: pod
    disposable: true
    volume:
      - name: workspace
        type: bind
        host: /home/atrawog/Sync/Atrapub/ecovoyage
    port:
      - "32718:2718"           # explicit pinned ports for stable bookmarks
      - "38080:8080"
      - "33000:3000"
      - "38000:8000"
      - "38001:8001"
      - "38002:8002"
      - "38090:8090"
```

CLI: `ov update versa/ecovoyage` ↔ `ov update versa -i ecovoyage`.

### Pinned-version pattern (Pattern B from /ov-core:deploy)

Run a specific image tag under an arbitrary deploy name (useful for
canaries, regression bisection, or holding back a specific version):

```yaml
deploy:
  versa-pinned-2026.131.2134:
    image: ghcr.io/overthinkos/versa:2026.131.2134  # exact ref, never re-resolved
    target: pod
    disposable: true
    port: [auto]
```

Container name: `ov-versa-pinned-2026.131.2134`. CLI:
`ov update versa-pinned-2026.131.2134`. The deploy key has no
relation to the image name; `ov update` always pulls the exact tag
declared in `image:`.

## MCP servers

This image publishes one MCP endpoint (registered by this plugin's
`.mcp.json`):

| Name | Container URL | Tool count | Skill |
|---|---|---:|---|
| marimo | `http://localhost:2718/mcp/server` | 10 (read-only inspection) | `/ov-versa:versa-mcp` |

The marimo MCP server name deliberately did NOT change in the
2026-05 image rename — it reflects the upstream software identity,
not OUR image identity. (Prior versions also published an `airflow`
MCP via `mcp-server-apache-airflow`; that wrapper was removed in
2026-05 because no Airflow-3 / `/api/v2` release of the upstream
package exists. Consumers drive Airflow via direct REST `/api/v2/`
calls instead.)

## R10 acceptance

```bash
ov update --build --force-seed versa
ov eval live versa         # → 93 passed · 0 failed · 0 skipped
```

The 11 new probes (added in the 2026-05 cutover; brings 82 → 93) all
live under the `image.versa.deploy_eval` block in `overthink.yml` so
they run AFTER every per-layer eval section:

| probe id | what it checks |
|---|---|
| `versa-notebook-export` | end-to-end `marimo export ipynb` — triggers all 6 self-authored DAGs and renders every cell server-side (the single highest-value probe; 600s timeout) |
| `versa-notebook-size` | rendered `.ipynb` is ≥100 KB (catches empty-cell renders) |
| `versa-martin-monaco` | TileJSON for the canonical `monaco` source returns 200 + body contains `tilejson` + `vector_layers` |
| `versa-martin-monaco-gpqtiles` | same for `monaco-gpqtiles` |
| `versa-martin-monaco-duckdb-mvt` | same for `monaco-duckdb-mvt` |
| `versa-martin-monaco-duckdb-freestiler` | same for `monaco-duckdb-freestiler` |
| `versa-versatiles-shortbread-tile` | versatiles serve returns a non-empty PBF tile for the shortbread output (HEAD content-length probe) |
| `versa-artifact-parquet` | post-DAG `monaco.parquet` ≥500 KB |
| `versa-artifact-pmtiles` | post-DAG `monaco.pmtiles` ≥1 MB |
| `versa-artifact-gtfs` | post-DAG `monaco.gtfs.zip` ≥500 KB |

End-to-end notebook test (executes all 13 cells via marimo's own
export):

```bash
podman exec ov-versa /home/user/.pixi/envs/default/bin/marimo \
  export ipynb /workspace/notebooks/osm-monaco-viz.py \
  --include-outputs --sort topological -o /tmp/notebook-run.ipynb -f
```

Expected outputs (verified end-to-end):

- both DAGs reach `state: success`
- OSM parquet ~824 KB at `/workspace/tiles/work/monaco.parquet`
- pmtiles ~2.6 MB at `/workspace/tiles/pmtiles/monaco.pmtiles`
- GTFS zip ~770 KB at `/workspace/gtfs/raw/monaco.gtfs.zip`
- 98 stops, 15 routes, 5846 trips in `/workspace/gtfs/parquet/`
- streets MapLibre HTML embeds martin URL, mapterhorn DEM, hillshade,
  terrain, sky, TerrainControl
- transit folium HTML embeds 98 `circleMarker` calls
- neither folium map carries the "Make this Notebook Trusted" wrapper

## Known gotchas

- **User deploy.yml needs `image:` field.** `~/.config/ov/deploy.yml`
  versa entry must include `image: versa` — otherwise
  `ov update --build` fails with "deploy has no 'image:' field".
- **Marimo session persistence reorders cells.** When a marimo session
  is open and the file changes (e.g. after `--force-seed`), marimo
  re-persists with its session's cell order. Surgical reset:
  `supervisorctl stop marimo && cp /data/workspace/notebooks/osm-monaco-viz.py /workspace/notebooks/ && supervisorctl start marimo`.
- **Martin caches pmtiles mtime at startup.** Re-run a DAG that
  rewrites `monaco.pmtiles` → martin returns 500/204 forever until
  restart. The OSM DAG's `reload_martin` task handles this automatically;
  see `/ov-versa:osm-tools-layer`.
- **Notebook needs `image:` field too.** Already covered above —
  same point worth emphasizing.

## Cross-references

- `/ov-versa:versa-layer` — marimo runtime layer (pixi env, service)
- `/ov-versa:versa-mcp` — marimo MCP server tool catalog
- `/ov-versa:airflow-layer` — Airflow 3.x compat findings
- `/ov-versa:notebook-osm` — the dual-DAG OSM+GTFS notebook
- `/ov-versa:maputnik-layer` — Vite `--base=/` build pattern
- `/ov-versa:osm-tools-layer` — martin reload pattern
- `/ov-versa:debug-tools-layer` — debug toolkit
- `/ov-eval:eval` — eval verbs used by the 83 probes
- `/ov-core:deploy` — env block authoring + cross-pod topology
- `/ov-internals:disposable` — `disposable: true` semantics
