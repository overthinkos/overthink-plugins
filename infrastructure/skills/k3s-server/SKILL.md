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
| Install files | `charly.yml`, `task:`, `service:`, `artifact:`, `secret_require:` |
| Depends on | `/charly-infrastructure:k3s` |
| Service | `k3s.service` (system scope, enabled) |

## What this layer does

1. Reads `K3S_CLUSTER_TOKEN` from the credential store (`secret_require:`).
2. Writes `/etc/rancher/k3s/config.yaml` with:
   - `token:` — the pre-shared cluster token
   - `write-kubeconfig-mode: "0644"` — so the operator can scp the kubeconfig back
   - `tls-san:` — `${K3S_SERVER_HOSTNAME}` (or `hostname -n` fallback)
   - `disable: []` — **explicitly empty**, so ServiceLB, Traefik v2, and
     local-path-provisioner all install as default k3s addons.
3. Emits `/etc/systemd/system/k3s.service` running `k3s server`.
4. After setup, the runtime publishes `/etc/rancher/k3s/k3s.yaml` back to
   the operator via the new `artifact:` layer-schema feature. The
   retrieved file lands at `~/.cache/charly/clusters/<deploy_name>/kubeconfig.yaml`
   with `127.0.0.1` rewritten to `${K3S_SERVER_HOSTNAME}` so the operator
   can `kubectl` the cluster from their machine.
5. The `K3sPostProvision` hook (Go, runs after artifact retrieval)
   merges the kubeconfig into `~/.kube/config` under context
   `<deploy_name>` and writes a matching ClusterProfile to
   `~/.config/charly/clusters/<deploy_name>.yaml` with `ingress.class=traefik`
   and `storage.class_default=local-path`.

## Operator setup — none required (auto-generated)

`K3S_CLUSTER_TOKEN` auto-generates on first deploy. The resolver
(`charly/layer_secrets.go` — `ensureLayerSecret`) detects the missing
`secret_require:` entry, generates a 32-byte hex token via
`generateAndStoreSecret`, and persists it to the active credential
backend (keyring / config-file fallback). Every subsequent
`k3s-server` and `k3s-agent` deploy reads the same persisted value —
zero operator setup, server and agents automatically share the token.

**Override** with a specific value (uncommon — only when reproducing a
specific cluster identity, e.g., disaster recovery):

```bash
charly secrets set charly/secret/K3S_CLUSTER_TOKEN $(openssl rand -hex 32)
```

**Retrieve** the auto-generated token (for debugging or
out-of-band agent join):

```bash
charly secrets get charly/secret K3S_CLUSTER_TOKEN
```

## Usage

```yaml
# charly.yml
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
charly vm create k3s-srv
charly deploy add vm:k3s-srv
# → kubeconfig auto-retrieved + ClusterProfile written
kubectl --context k3s-srv get nodes
charly eval k8s addons --cluster k3s-srv
```

## Tests

Build-scope:
- `/etc/rancher/k3s/config.yaml` exists, mode 0600.
- `/etc/systemd/system/k3s.service` exists.

Deploy-scope (using the new `charly eval k8s` verb — see `/charly-kubernetes:eval-k8s`):
- `k8s: wait-nodes` — at least 1 node Ready.
- `k8s: ingressclass` — `traefik` present.
- `k8s: storageclass` — `local-path` present.
- `k8s: addons` — Traefik + ServiceLB + local-path-provisioner all Ready.

## Related Layers
- `/charly-infrastructure:k3s` — Base layer installing the k3s binary (required dep)
- `/charly-infrastructure:k3s-agent` — Worker nodes joining this server
- `/charly-coder:kubernetes-layer` — `kubectl`/`helm` on the operator side (not needed in the cluster)
- `/charly-kubernetes:eval-k8s` — The test verb used by this layer's deploy-scope checks
