---
name: notebooks-llm-on-supercomputers
description: |
  LLMs on Supercomputers course notebook collection (TU Wien AI Factory Austria).
  15 Jupyter notebooks covering prompt engineering, RAG, and fine-tuning.
  Data-only layer — no packages, no services, no dependencies.
  Use when working with the LLM course notebooks, LangChain tutorials, or RAG examples.
---

# notebooks-llm-on-supercomputers -- LLM course data layer

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | *(none)* |
| Packages | *(none)* |
| Services | *(none)* |
| Volumes | `workspace` -> `~/workspace` (from jupyter-colab) |
| Data | `data/llms_on_supercomputers` -> `workspace` volume, dest: `llms_on_supercomputers` |
| Install files | *(none)* |

## How It Works

This is a **data layer** — it uses the `data:` field in `layer.yml` to map a directory of notebooks to a named volume with a subdirectory destination:

```yaml
info: "LLMs on Supercomputers course notebooks (TU Wien AI Factory Austria)"

data:
  - src: data/llms_on_supercomputers
    volume: workspace
    dest: llms_on_supercomputers
```

At build time, the contents are staged into `/data/workspace/llms_on_supercomputers/` inside the image. At deploy time, `ov config` or `ov update` provisions them into the workspace volume.

## Included Notebooks

15 Jupyter notebooks organized in 4 categories:

### D0 — Setup (1 notebook)

| Notebook | Topic |
|----------|-------|
| `D0_00_Bazzite_AI_Setup.ipynb` | Environment setup, GPU verification, Ollama connectivity |

### D1 — Prompt Engineering (6 notebooks)

| Notebook | Topic | Libraries |
|----------|-------|-----------|
| `D1_01_Prompting_with_LangChain.ipynb` | LangChain basics, local + Ollama models | LangChain, OpenAI, HuggingFace |
| `D1_02_Prompt_templates_and_parsing.ipynb` | Prompt templates, few-shot, structured output | LangChain, OpenAI |
| `D1_05_Chaining.ipynb` | Chain composition and routing | LangChain |
| `D1_08_LLM_Evaluation.ipynb` | LLM evaluation with evidently.ai | ollama, evidently |
| `D1_09_LLM_as_a_Judge.ipynb` | LLM-as-a-Judge evaluation pattern | OpenAI, evidently |
| `D1_10_Prompt_Optimization.ipynb` | Prompt optimization techniques | OpenAI, evidently |

### D2 — Retrieval Augmented Generation (2 notebooks)

| Notebook | Topic | Libraries |
|----------|-------|-----------|
| `D2_01_rag_with_basic_tools.ipynb` | RAG with pandas DataFrame as vector store | OpenAI, numpy, pandas |
| `D2_02_rag_with_langchain_and_chromadb.ipynb` | RAG pipeline with ChromaDB | LangChain, ChromaDB, OllamaEmbeddings |

### D3 — Fine-tuning on One GPU (6 notebooks)

| Notebook | Topic | Libraries |
|----------|-------|-----------|
| `D3_01_Transformer_Architecture.ipynb` | Transformer theory (no LLM calls) | PyTorch |
| `D3_02_Finetuning_LLM_with_PyTorch.ipynb` | Manual fine-tuning loop | PyTorch, transformers |
| `D3_03_Finetuning_LLM_with_Huggingface.ipynb` | Fine-tuning with HF Trainer | transformers, datasets |
| `D3_04_Quantization.ipynb` | Model quantization techniques | bitsandbytes, transformers |
| `D3_05_PEFT.ipynb` | Parameter-Efficient Fine-Tuning (LoRA) | peft, transformers |
| `D3_06_Unsloth.ipynb` | Fine-tuning with Unsloth | unsloth |

## Supporting Files

| File | Purpose |
|------|---------|
| `notebooks.yaml` | Structured catalog with subcategories |
| `LICENSE` | Apache 2.0 (TU Wien AI Factory Austria) |
| `simplified_output.json` | Sample data for D1_02 prompt templates |
| `datasets/booking_queries_dataset.csv` | Booking classification data |
| `datasets/code_review_dataset.csv` | Code review data |
| `datasets/health_and_fitness_qna.csv` | Health Q&A data |

## Network Connectivity

9 notebooks (D0, D1, D2 series) connect to Ollama at `http://ov-ollama:11434` via Podman DNS on the `ov` bridge network. The D3 fine-tuning notebooks work locally with GPU — no Ollama needed.

## Notebook Compatibility Notes

### ollama Python library

The same `importlib.reload(ollama)` pattern from `/ov-layers:ollama-notebooks` is applied to all cleanup cells that use `import ollama` for model unloading.

### External services

- D1_08, D1_09, D1_10 reference `https://app.evidently.cloud` for LLM evaluation dashboards (optional, notebooks work without it)

### Default models

- D1 series: `hf.co/NousResearch/Nous-Hermes-2-Mistral-7B-DPO-GGUF:Q4_K_M`
- D2 series: `llama3.2:latest`

## Used In Images

- `/ov-images:jupyter-colab-ml-finetuning`

## Related Skills

- `/ov:layer` — data field documentation and layer authoring rules
- `/ov:config` — data provisioning during `ov config` setup
- `/ov-layers:finetuning-notebooks` — sibling data layer (Unsloth fine-tuning notebooks)
- `/ov-layers:ollama-notebooks` — sibling data layer (Ollama API tutorials)
- `/ov-layers:notebook-templates` — sibling data layer (starter notebooks)
- `/ov-images:ollama` — the Ollama server image (must be running for D0-D2 notebooks)
- `/ov-images:jupyter-colab-ml-finetuning` — the image that includes this layer

## When to Use This Skill

Use when the user asks about:

- The notebooks-llm-on-supercomputers layer or its contents
- LangChain prompt engineering tutorials
- RAG examples with ChromaDB
- The TU Wien LLM course notebooks
- Fine-tuning notebooks (PyTorch, HuggingFace, PEFT, Unsloth)
