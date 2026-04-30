---
name: k3s-server
description: |
  k3s control-plane (server) node with ServiceLB, Traefik v2, and local-path-provisioner enabled by default.
  Publishes kubeconfig back to the operator via layer artifacts and registers a ClusterProfile on first boot.
---

# k3s-server -- k3s control-plane node

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `tasks:`, `service:`, `artifacts:`, `secret_requires:` |
| Depends on | `/ov-foundation:k3s` |
| Service | `k3s.service` (system scope, enabled) |

## What this layer does

1. Reads `K3S_CLUSTER_TOKEN` from the credential store (`secret_requires:`).
2. Writes `/etc/rancher/k3s/config.yaml` with:
   - `token:` — the pre-shared cluster token
   - `write-kubeconfig-mode: "0644"` — so the operator can scp the kubeconfig back
   - `tls-san:` — `${K3S_SERVER_HOSTNAME}` (or `hostname -n` fallback)
   - `disable: []` — **explicitly empty**, so ServiceLB, Traefik v2, and
     local-path-provisioner all install as default k3s addons.
3. Emits `/etc/systemd/system/k3s.service` running `k3s server`.
4. After setup, the runtime publishes `/etc/rancher/k3s/k3s.yaml` back to
   the operator via the new `artifacts:` layer-schema feature. The
   retrieved file lands at `~/.cache/ov/clusters/<deploy_name>/kubeconfig.yaml`
   with `127.0.0.1` rewritten to `${K3S_SERVER_HOSTNAME}` so the operator
   can `kubectl` the cluster from their machine.
5. The `K3sPostProvision` hook (Go, runs after artifact retrieval)
   merges the kubeconfig into `~/.kube/config` under context
   `<deploy_name>` and writes a matching ClusterProfile to
   `~/.config/ov/clusters/<deploy_name>.yaml` with `ingress.class=traefik`
   and `storage.class_default=local-path`.

## One-time operator setup

Before the FIRST k3s-server deploy, the operator sets the pre-shared
cluster token once:

```bash
ov secrets set ov/secret/K3S_CLUSTER_TOKEN $(openssl rand -hex 32)
```

Every `k3s-server` and `k3s-agent` deploy reads the same secret — no
per-deploy token dance.

## Usage

```yaml
# overthink.yml
vm:
  k3s-srv:
    source: { kind: cloud_image, url: "…" }
    disposable: true
    ram: 4G
    cpus: 2

deployments:
  images:
    "vm:k3s-srv":
      target: vm
      vm_source: k3s-srv
      add_layers: [k3s-server]
      env:
        - K3S_SERVER_HOSTNAME=k3s-srv.lan  # optional but recommended
```

```bash
ov vm create k3s-srv
ov deploy add vm:k3s-srv
# → kubeconfig auto-retrieved + ClusterProfile written
kubectl --context k3s-srv get nodes
ov eval k8s addons --cluster k3s-srv
```

## Tests

Build-scope:
- `/etc/rancher/k3s/config.yaml` exists, mode 0600.
- `/etc/systemd/system/k3s.service` exists.

Deploy-scope (using the new `ov eval k8s` verb — see `/ov-advanced:eval-k8s`):
- `k8s: wait-nodes` — at least 1 node Ready.
- `k8s: ingressclass` — `traefik` present.
- `k8s: storageclass` — `local-path` present.
- `k8s: addons` — Traefik + ServiceLB + local-path-provisioner all Ready.

## Related Layers
- `/ov-foundation:k3s` — Base layer installing the k3s binary (required dep)
- `/ov-foundation:k3s-agent` — Worker nodes joining this server
- `/ov-coder:kubernetes-layer` — `kubectl`/`helm` on the operator side (not needed in the cluster)
- `/ov-advanced:eval-k8s` — The test verb used by this layer's deploy-scope checks
