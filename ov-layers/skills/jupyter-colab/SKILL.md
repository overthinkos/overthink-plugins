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
| Ports | 8888 |
| Service | `jupyter-colab` (supervisord) |
| Volume | `notebooks` at `~/notebooks` |
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

The `notebooks` volume is mounted at `~/notebooks` (NOT `/workspace`). This avoids a collision with `ov`'s hardcoded workspace mount which always targets `/workspace`.

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

## Implementation Notes

- The supervisord service command calls `jupyter lab` directly, not via `pixi run`. Pixi multi-stage builds copy only `~/.pixi/envs/` to the final image — `pixi.toml` is not present at runtime.
- The `start-jupyter` pixi task in `pixi.toml` is for build-time verification only.

## Used In Images

- `/ov-images:jupyter-colab`

## Related Layers

- `/ov-layers:jupyter` -- GPU-accelerated Jupyter with ML libraries (heavy variant)
- `/ov-layers:supervisord` -- process manager dependency
- `/ov-layers:python` -- Python runtime (transitive via supervisord)

## When to Use This Skill

Use when the user asks about:

- Real-time collaborative Jupyter notebooks
- jupyter-collaboration or Y-CRDT setup
- Lightweight Jupyter without GPU/ML dependencies
- Port 8888 service (check if GPU variant intended)
- The `jupyter-colab` layer
