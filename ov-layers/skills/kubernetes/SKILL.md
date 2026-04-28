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
| Install files | `layer.yml`, `tasks:` |

## Packages

RPM: `kubernetes-client`, `helm`

## Cross-distro coverage

`rpm:` (Fedora), `pac:` (Arch — `kubectl` + `helm` from `extra`), `deb:` — adds two upstream apt repos: `https://pkgs.k8s.io/core:/stable:/v1.30/deb/` for `kubectl` and `https://baltocdn.com/helm/stable/debian/all` for `helm`. Both use signed-by GPG keys. The flat-repo `pkgs.k8s.io` URL ends in `/` — supported by `build.yml`'s deb install template (the trailing-slash suite special case added during Phase 3).

## Usage

```yaml
# image.yml
my-devops:
  layers:
    - kubernetes
```

## Used In Images

- `bazzite-ai` (disabled)

## Related Layers
- `/ov-layers:docker-ce` — common companion for local container builds
- `/ov-layers:devops-tools` — provides kubectx/kubens and cloud CLIs
- `/ov-layers:dev-tools` — typically paired in DevOps images

## Related Commands
- `/ov:build` — installs kubectl and helm RPMs during image build
- `/ov:shell` — run kubectl/helm interactively against external clusters

## When to Use This Skill

Use when the user asks about:

- Kubernetes tools in containers
- kubectl or Helm setup
- The `kubernetes` layer

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
