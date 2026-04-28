---
name: google-cloud
description: |
  Google Cloud SDK providing gcloud, gsutil, and bq CLI tools.
  Use when working with Google Cloud Platform, GCP services, or cloud SDK configuration.
---

# google-cloud -- Google Cloud SDK

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `tasks:` |

## Usage

```yaml
# image.yml
my-image:
  layers:
    - google-cloud
```

## Used In Images

- `/ov-images:bazzite-ai` (disabled)

## Related Layers

- `/ov-layers:google-cloud-npm` -- additional GCP npm tools (firebase-tools, gemini-cli)

## When to Use This Skill

Use when the user asks about:

- Google Cloud SDK installation
- gcloud, gsutil, or bq commands
- GCP authentication or configuration
- Google Cloud Platform services

## Author + Test References

- `/ov:layer` — layer authoring reference (tasks, vars, env_provides, tests block syntax)
- `/ov:eval` — declarative testing framework for the `tests:` block
