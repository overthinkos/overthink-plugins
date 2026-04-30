---
name: llama-cpp
description: |
  llama.cpp prebuilt binaries and GGUF conversion tools.
  Use when working with llama.cpp, GGUF model conversion, or llama-quantize/llama-cli.
---

# llama-cpp -- llama.cpp prebuilt binaries and GGUF tools

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | None |
| Ports | — |
| Service | — |
| Install files | `layer.yml`, `tasks:` |

## What It Installs

Downloads the latest llama.cpp release from GitHub into `~/llama.cpp`:

**Binaries:** `llama-quantize`, `llama-cli`, shared libraries (`lib*.so*`)

**Python tools:** `convert_hf_to_gguf.py`, `gguf-py` package (from source tarball)

## Environment

| Variable | Value | Purpose |
|----------|-------|---------|
| `LLAMA_CPP_PATH` | `~/llama.cpp` | Location of llama.cpp binaries |
| `PATH` (appended) | `~/llama.cpp` | Makes llama-quantize/llama-cli available |

## Design: Tier 1 "Post-install" Layer

This layer has **no pixi.toml** and **no depends**. It downloads prebuilt binaries and sets environment variables. It is designed to be composed into environment-owning layers (Tier 2) via the `layers:` field.

The user-phase tasks run after the pixi environment is established by the parent layer. The `gguf` Python package (for programmatic GGUF access) is declared in the parent layer's pixi.toml, not here.

## Used In Layers

- `/ov-foundation:python-ml` — via `layers: [llama-cpp]`
- `/ov-jupyter:jupyter-ml` — via `layers: [llama-cpp, unsloth]`
- `/ov-jupyter:unsloth-studio` — via `layers: [llama-cpp, unsloth]`

## Related Layers

- `/ov-jupyter:unsloth` — Fine-tuning (depends on llama.cpp for GGUF conversion)

## Used In Images

- `/ov-foundation:python-ml` (via `python-ml` metalayer)
- `/ov-immich:immich-ml` (via `python-ml` metalayer)
- `/ov-jupyter:jupyter-ml` (via `jupyter-ml` metalayer)
- `/ov-jupyter:jupyter-ml-notebook` (via `jupyter-ml` metalayer)
- `/ov-jupyter:unsloth-studio` (via `unsloth-studio` metalayer)

## When to Use This Skill

Use when the user asks about:

- llama.cpp binaries inside containers
- GGUF model conversion (convert_hf_to_gguf.py)
- llama-quantize or llama-cli
- The `LLAMA_CPP_PATH` environment variable

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
