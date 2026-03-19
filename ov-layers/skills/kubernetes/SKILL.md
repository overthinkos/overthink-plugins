---
name: kubernetes
description: |
  Kubernetes client tools: kubectl and Helm package manager.
  Use when working with Kubernetes, kubectl, or Helm charts.
---

# kubernetes -- kubectl and Helm

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `root.yml` |

## Packages

RPM: `kubernetes-client`, `helm`

## Usage

```yaml
# images.yml
my-devops:
  layers:
    - kubernetes
```

## Used In Images

- `bazzite-ai` (disabled)

## When to Use This Skill

Use when the user asks about:

- Kubernetes tools in containers
- kubectl or Helm setup
- The `kubernetes` layer
