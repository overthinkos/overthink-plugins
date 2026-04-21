---
name: kubernetes
description: |
  MUST be invoked before any work involving: `ov deploy add --target kubernetes`, `ov deploy from-image`, Kustomize manifest generation, cluster profiles, K8s deployments, `kubernetes:` block in deploy spec, or OCI-label capabilities.
---

# Kubernetes — Deploying Overthink Images to K8s Clusters

## Overview

Overthink can deploy built images to a Kubernetes cluster by emitting a Kustomize `base/` + `overlays/` tree. The deployment schema stays **target-agnostic** — authors describe *what the workload needs* (kind, replicas, resources, exposure, storage, probes); a per-cluster **cluster profile** file supplies the K8s-specific knobs (storage class, ingress class, cert issuer, secret backend).

Every image's runtime contract is baked into OCI labels at build time, so **a K8s deploy is possible without access to `overthink.yml`** — the `ov deploy from-image` verb reads capabilities from the pushed image alone.

## Quick reference

| Action | Command | Description |
|---|---|---|
| Add K8s deploy | `ov deploy add <name> <ref> --target kubernetes` | Read ImageConfig + deployment + cluster profile; emit `.overthink/k8s/<name>/` Kustomize tree |
| Source-less deploy | `ov deploy from-image <registry/name:tag> [name] --cluster <name>` | Deploy from OCI labels only — no `overthink.yml` needed (see Part F.10) |
| Sync to cluster | `ov deploy sync <name>` | `kubectl apply -k .overthink/k8s/<name>/overlays/default` |
| Show generated manifests | `ov deploy show <name>` | `kubectl kustomize …` — see what would apply |
| Delete K8s deploy | `ov deploy del <name>` | Remove overlay dir; base stays if other instances reference it |

## The three-layer model

| Concern | Schema slot | OCI label home |
|---|---|---|
| **Build** — what goes INTO the image | `image.build:` (or legacy `ImageConfig`) | no (consumed at build) |
| **Capabilities** — image's runtime contract | `image.capabilities:` (or layer rollups) | **yes** — every field under `org.overthinkos.*` |
| **Deployment** — how to run the image | `overthink.yml:deployments.<name>` + `~/.config/ov/deploy.yml` overlay | no |

The completeness invariant: every exported field on `ImageMetadata`/`Capabilities` has a `CapabilityLabelMap` entry. A compile-time test enforces this — a new capability field without a label mapping fails the build. See `ov/capabilities.go`.

## Deployment schema — target-agnostic fields

```yaml
deployments:
  images:
    openclaw:
      target: kubernetes
      kind: service                   # service | daemon | batch | scheduled | oneshot
      replicas: 3
      restart: always                 # always | on-failure | never (honored on Pod/Job/CronJob)
      resources:
        cpu_request: "500m"
        memory_request: 512Mi
      security:
        memory_max: 2Gi               # → resources.limits.memory
        cpus: "1.5"                   # → resources.limits.cpu
      expose:
        host: openclaw.example.com
        path: /
        tls: true                     # → cert-manager annotation from cluster profile
      storage:
        - { name: data, size: 20Gi, class_hint: fast, access: single-writer }
      probes:
        liveness:  { http: { path: /healthz, port: 8080 } }
        readiness: { http: { path: /ready,   port: 8080 } }
      kubernetes:
        cluster: production           # pointer to ~/.config/ov/clusters/production.yaml
        namespace: apps               # optional override of cluster profile default
        patches: []                   # escape hatch: strategic / JSON6902 patches
        raw: []                       # escape hatch: paths to raw manifests included verbatim
```

**Workload kind heuristic** (inside `ov/k8s_generate.go`):

```
kind: service   + storage: []       → Deployment
kind: service   + storage: [...]    → StatefulSet  (auto volumeClaimTemplates)
kind: daemon                        → DaemonSet
kind: batch                         → Job
kind: scheduled (+schedule:)        → CronJob
kind: oneshot                       → Pod
```

Explicit override: `kubernetes.workload: Deployment` (rare — prefer `kind:`).

## Cluster profile

One file per cluster. Lives at `~/.config/ov/clusters/<name>.yaml` (per-user) or `clusters/<name>.yaml` (in-repo, discoverable via `discover.clusters:`). The **only** place cluster-specific K8s knobs live.

```yaml
# ~/.config/ov/clusters/production.yaml
version: 1
kind: cluster-profile
name: production
kubeconfig_context: gke_prod_us-east1
admission_policy: restricted         # restricted | baseline | privileged
default_namespace: apps

storage:
  class_default: fast-ssd-retain
  class_fast: fast-ssd
  class_cheap: hdd-delete
  class_encrypted: fast-ssd-luks
  access_mode_default: ReadWriteOnce

ingress:
  enabled: true
  class: nginx
  cert_issuer: letsencrypt-prod      # cert-manager ClusterIssuer
  path_type_default: Prefix

secrets:
  backend: external-secrets          # external-secrets | sealed-secrets | raw
  store: vault-prod
  prefix: prod/

images:
  pull_policy: IfNotPresent
  pull_secrets: [regcred-prod]

pod_defaults:
  priority_class: standard
  tolerations: []
  node_selector: {}

defaults:
  labels: { managed-by: overthink }
```

New cluster = write a new profile; zero deployment-spec changes.

## Generator output

```
.overthink/k8s/<deployment-name>/
├── base/
│   ├── kustomization.yaml           # commonLabels, resources: […]
│   ├── deployment.yaml              # or statefulset.yaml / daemonset.yaml / job.yaml / cronjob.yaml / pod.yaml
│   ├── service.yaml
│   ├── pvc-<name>.yaml              # per storage entry (Deployment/DaemonSet); StatefulSet uses volumeClaimTemplates
│   ├── ingress.yaml                 # when expose.host set AND cluster profile ingress.enabled
│   └── raw/                         # copied from kubernetes.raw:
└── overlays/
    └── <instance>/                  # "default" for bare name; "prod" for image/prod
        └── kustomization.yaml       # namespace override + patches from kubernetes.patches:
```

Apply: `kubectl apply -k .overthink/k8s/<name>/overlays/<instance>` (or `ov deploy sync <name>`).

## Source-less deploy — `ov deploy from-image`

Proves the self-contained image invariant: a deploy pipeline with **no access to `overthink.yml`** can still produce a correct Kustomize tree.

```bash
# On a machine that doesn't have the source repo:
ov deploy from-image quay.io/myorg/openclaw:v2 openclaw \
    --target kubernetes --cluster production --namespace apps
# Reads: OCI labels (capabilities) + ~/.config/ov/clusters/production.yaml
#        + ~/.config/ov/deploy.yml (if present, for per-machine overrides)
# Emits: .overthink/k8s/openclaw/base/ + overlays/default/
ov deploy sync openclaw                   # kubectl apply -k ...
```

## Relevant code

- `ov/k8s_config.go` — `K8sDeployConfig` + `ClusterProfile` + `LoadClusterProfile`
- `ov/k8s_target.go` — `K8sDeployTarget` (fourth DeployTarget alongside OCI / container / host)
- `ov/k8s_generate.go` — `GenerateK8sKustomize` + workload-kind heuristic + Ingress/PVC emission
- `ov/k8s_deploy_from_image.go` — `DeployFromImage`
- `ov/capabilities.go` — `Capabilities` (alias of `ImageMetadata`) + `CapabilityLabelMap` + completeness check

## Related skills

- `/ov:deploy` — unified `ov deploy add`/`del` verb; K8s is one of three targets
- `/ov-dev:capabilities` — OCI label contract the K8s generator reads from
- `/ov-dev:install-plan` — shared IR across build + deploy targets
