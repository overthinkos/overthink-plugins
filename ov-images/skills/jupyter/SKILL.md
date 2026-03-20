---
name: jupyter
description: |
  Jupyter notebook server with CUDA GPU support and ML libraries.
  Runs as a supervisord service on port 8888. Use when working with
  Jupyter notebooks, ML training, or GPU-accelerated Python.
---

# jupyter

GPU-accelerated Jupyter notebook server with ML libraries and llama.cpp.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia |
| Layers | jupyter |
| Platforms | linux/amd64 |
| Ports | 8888 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` → `nvidia` (CUDA base)
2. `pixi` → `python` → `supervisord` (transitive)
3. `jupyter` — Jupyter server, ML pixi packages, llama.cpp

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 8888 | Jupyter notebook | HTTP |

## Quick Start

```bash
ov build jupyter
ov enable jupyter
ov start jupyter
# Open http://localhost:8888
```

## Key Layers

- `/ov-layers:jupyter` — Jupyter server + ML pixi environment
- `/ov-layers:cuda` — GPU support (via nvidia base)

## Related Images

- `/ov-images:nvidia` — parent (GPU without Jupyter)
- `/ov-images:python-ml` — ML libraries without Jupyter server

## Verification

After `ov start`:
- `ov status jupyter` — container running
- `ov service status jupyter` — all supervisord services RUNNING
- `curl -s -o /dev/null -w '%{http_code}' http://localhost:8888` — Jupyter HTTP returns 200

## When to Use This Skill

Use when the user asks about the jupyter image, running Jupyter notebooks, ML development in containers, or GPU-accelerated Python notebooks.
