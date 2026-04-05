---
name: finetuning-notebooks
description: |
  Unsloth fine-tuning notebook collection provisioned into the workspace volume at deploy time.
  Data-only layer — no packages, no services, no dependencies.
  Use when working with finetuning-notebooks, Unsloth training notebooks, or unsloth-studio data provisioning.
---

# finetuning-notebooks -- Unsloth fine-tuning notebook data layer

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | *(none)* |
| Packages | *(none)* |
| Services | *(none)* |
| Volumes | `workspace` -> `~/workspace` (from unsloth-studio) |
| Data | `data/finetuning` -> `workspace` volume, dest: `finetuning` |
| Install files | *(none)* |

## How It Works

This is a **data layer** — it uses the `data:` field in `layer.yml` to map a directory of notebooks to a named volume with a subdirectory destination:

```yaml
info: "Unsloth fine-tuning notebook collection for unsloth-studio"

data:
  - src: data/finetuning
    volume: workspace
    dest: finetuning
```

At build time, the contents of `data/finetuning/` are staged into `/data/workspace/finetuning/` inside the image.

At deploy time, when the workspace volume is configured as a bind mount (`ov config --bind workspace`), `ov config` copies the staged data into the host-backed volume directory at `<workspace>/finetuning/`. This seeds the volume with ready-to-use training notebooks.

The `dest: finetuning` field places the notebooks in a subdirectory rather than the volume root, keeping the workspace organized alongside other content.

## Included Data

37 Jupyter notebooks + 1 manifest, organized by training category:

| Category | Notebooks | Models |
|----------|-----------|--------|
| 00-Setup | `00_Unsloth_Setup.ipynb` | *(general)* |
| 01-FastInference | 3 notebooks | Llama, Qwen, Qwen_Think |
| 02-Vision Training | 2 notebooks | Ministral, Pixtral |
| 03-SFT Training | 5 notebooks | Ministral (text + vision), Pixtral, Qwen, Qwen_Think |
| 04-GRPO Training | 5 notebooks | Ministral (text + vision), Pixtral, Qwen, Qwen_Think |
| 05-DPO Training | 2 notebooks | Qwen, Qwen_Think |
| 06-Reward Training | 2 notebooks | Qwen, Qwen_Think |
| 07-RLOO Training | 5 notebooks | Ministral (text + vision), Pixtral, Qwen, Qwen_Think |
| 08-QLoRA | 12 notebooks | Ministral, Qwen_Think (alpha scaling, continual learning, rank comparison, multi-adapter, quantization comparison, target modules) |

**Manifest:** `notebooks.yaml` — structured catalog of all notebooks with metadata.

## Usage

```yaml
# images.yml
unsloth-studio:
  layers:
    - unsloth-studio
    - finetuning-notebooks
    # ... other layers
```

```bash
# Deploy with bind-backed workspace volume
ov config unsloth-studio --bind workspace

# Notebooks are seeded at <workspace>/finetuning/ on first config
ov start unsloth-studio
# Open http://localhost:8888 → navigate to finetuning/
```

## Used In Images

- `/ov-images:unsloth-studio`

## Related Skills

- `/ov:layer` -- data field documentation and layer authoring rules
- `/ov:config` -- data provisioning during `ov config` setup
- `/ov:deploy` -- volume backing configuration (bind, named, encrypted)
- `/ov-layers:unsloth-studio` -- the Tier 2 meta-layer that owns the workspace volume
- `/ov-layers:notebook-templates` -- sibling data layer pattern (starter notebooks for jupyter-colab)
- `/ov-images:unsloth-studio` -- the image that includes this layer

## When to Use This Skill

Use when the user asks about:

- The finetuning-notebooks layer or its contents
- Unsloth fine-tuning notebook templates or training workflows
- How training notebooks get provisioned into the workspace volume
- The `data:` field with `dest:` subdirectory mapping
- Which models and training methods are covered by the notebook collection
