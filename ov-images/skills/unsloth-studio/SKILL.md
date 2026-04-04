---
name: unsloth-studio
description: |
  Unsloth Studio fine-tuning web UI with CUDA GPU support, vLLM inference, and llama.cpp.
  Runs as a supervisord service on ports 8888 (Studio) and 8000 (vLLM API).
  MUST be invoked before building, deploying, configuring, or troubleshooting the unsloth-studio image.
---

# unsloth-studio

Unsloth Studio web UI for LLM fine-tuning with GPU acceleration.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia |
| Layers | agent-forwarding, unsloth-studio |
| Platforms | linux/amd64 |
| Ports | 8888, 8000 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` -> `nvidia` (CUDA base)
2. `pixi` -> `python` -> `supervisord` (transitive)
3. `unsloth` -- Fine-tuning ML environment, vLLM, llama.cpp
4. `unsloth-studio` -- Studio web UI service

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 8888 | Unsloth Studio UI | HTTP |
| 8000 | vLLM API server | HTTP |

## Volumes

| Name | Path | Purpose |
|------|------|---------|
| models | ~/.cache/huggingface | HuggingFace model cache |
| workspace | ~/workspace | Training data and outputs |

## Quick Start

```bash
ov build unsloth-studio
ov config unsloth-studio
ov start unsloth-studio
# Open http://localhost:8888
```

## Key Layers

- `/ov-layers:unsloth` -- Core fine-tuning environment (PyTorch, transformers, peft, trl, bitsandbytes, vLLM)
- `/ov-layers:unsloth-studio` -- Studio web UI service
- `/ov-layers:cuda` -- GPU support (via nvidia base)

## Related Images

- `/ov-images:nvidia` -- parent (GPU without Studio)
- `/ov-images:jupyter` -- alternative ML UI (Jupyter notebooks, shares port 8888)
- `/ov-images:python-ml` -- ML libraries without any UI

## Verification

After `ov start`:
- `ov status unsloth-studio` -- container running
- `ov service status unsloth-studio` -- all services RUNNING
- `curl -s -o /dev/null -w '%{http_code}' http://localhost:8888` -- Studio HTTP returns 200

## When to Use This Skill

**MUST be invoked** when the task involves the unsloth-studio image, LLM fine-tuning via web UI, or Unsloth Studio deployment. Invoke this skill BEFORE reading source code or launching Explore agents.
