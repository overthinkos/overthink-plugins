---
name: notebook-templates
description: |
  Starter notebook templates provisioned into the notebooks volume at deploy time.
  First data-only layer in the project — no packages, no services, no dependencies.
  Use when working with notebook-templates, data layers, or jupyter-colab initial content.
---

# notebook-templates -- Starter notebook data layer

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | *(none)* |
| Packages | *(none)* |
| Services | *(none)* |
| Volumes | `notebooks` -> `~/notebooks` |
| Data | `data/notebooks` -> `notebooks` volume |
| Install files | *(none)* |

## How It Works

This is a **data layer** — the first of its kind in the project. It uses the `data:` field in `layer.yml` to map a directory of files to a named volume:

```yaml
info: "Starter notebook templates for jupyter-colab"

data:
  - src: data/notebooks
    volume: notebooks
```

At build time, the contents of `data/notebooks/` are staged into `/data/notebooks/` inside the image.

At deploy time, when the volume is configured as a bind mount (`ov config --bind notebooks`), `ov config` copies the staged data from the image into the host-backed volume directory. This seeds the volume with starter content (e.g., `getting-started.ipynb`).

## Included Data

| File | Purpose |
|------|---------|
| `getting-started.ipynb` | Starter notebook for new jupyter-colab deployments |

## Usage

```yaml
# images.yml
jupyter-colab:
  layers:
    - notebook-templates
    # ... other layers
```

```bash
# Deploy with bind-backed notebooks volume
ov config jupyter-colab --bind notebooks

# Data is copied from image to host volume on first config
ov start jupyter-colab
```

## Used In Images

- `/ov-images:jupyter-colab`

## Related Skills

- `/ov:layer` -- data field documentation and layer authoring rules
- `/ov:config` -- data provisioning during `ov config` setup
- `/ov:deploy` -- volume backing configuration (bind, named, encrypted)
- `/ov-layers:jupyter-colab` -- the JupyterLab layer that consumes the notebooks volume
- `/ov-images:jupyter-colab` -- the image that includes this layer

## When to Use This Skill

Use when the user asks about:

- The notebook-templates layer or its contents
- Data layers and the `data:` field in `layer.yml`
- How starter content gets provisioned into volumes
- The `getting-started.ipynb` notebook
- How `ov config` seeds bind-backed volumes with image data
