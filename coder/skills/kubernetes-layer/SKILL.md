---
name: kubernetes-layer
description: |
  Kubernetes client tools: kubectl and Helm package manager.
  Use when working with Kubernetes, kubectl, or Helm charts.
---

# kubernetes -- kubectl and Helm

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `charly.yml`, `task:` |

## Packages

RPM: `kubernetes-client`, `helm`

## Cross-distro coverage

`rpm:` (Fedora), `pac:` (Arch — `kubectl` + `helm` from `extra`), `deb:` — adds two upstream apt repos: `https://pkgs.k8s.io/core:/stable:/v1.30/deb/` for `kubectl` and `https://baltocdn.com/helm/stable/debian/all` for `helm`. Both use signed-by GPG keys. The flat-repo `pkgs.k8s.io` URL ends in `/` — supported by `build.yml`'s deb install template (the trailing-slash suite special case added during Phase 3).

## Usage

```yaml
# charly.yml
my-devops:
  layers:
    - kubernetes
```

## Used In Images

- (none currently enabled)

## Related Layers
- `/charly-coder:docker-ce` — common companion for local container builds
- `/charly-coder:devops-tools` — provides kubectx/kubens and cloud CLIs
- `/charly-coder:dev-tools` — typically paired in DevOps images

## Related Commands
- `/charly-build:build` — installs kubectl and helm RPMs during image build
- `/charly-core:shell` — run kubectl/helm interactively against external clusters

## When to Use This Skill

Use when the user asks about:

- Kubernetes tools in containers
- kubectl or Helm setup
- The `kubernetes` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
