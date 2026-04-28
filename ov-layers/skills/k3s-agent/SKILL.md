---
name: k3s-agent
description: |
  k3s worker (agent) node — joins an existing k3s-server via pre-shared token.
  Fully declarative: same ov secrets set once + env K3S_SERVER_URL per agent deploy.
---

# k3s-agent -- k3s worker node

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `tasks:`, `service:`, `secret_requires:`, `env_requires:` |
| Depends on | `/ov-layers:k3s` |
| Service | `k3s-agent.service` (system scope, enabled) |

## What this layer does

1. Reads `K3S_CLUSTER_TOKEN` from the credential store (same secret the
   server consumes).
2. Requires `K3S_SERVER_URL` from deploy.yml env (e.g.,
   `https://k3s-srv.lan:6443`).
3. Writes `/etc/rancher/k3s/config.yaml` with `server:` and `token:`.
4. Emits `/etc/systemd/system/k3s-agent.service` running `k3s agent`.

No join-token handoff, no kubeconfig retrieval — agents only need the
server URL (declarative, known at author time) and the pre-shared token
(from the credential store).

## Usage

```yaml
# overthink.yml (assumes k3s-srv already up; see /ov-layers:k3s-server)
vm:
  k3s-ag1:
    source: { kind: cloud_image, url: "…" }
    disposable: true
    ram: 4G
    cpus: 2

deployments:
  images:
    "vm:k3s-ag1":
      target: vm
      vm_source: k3s-ag1
      add_layers: [k3s-agent]
      env:
        - K3S_SERVER_URL=https://k3s-srv.lan:6443
        # K3S_CLUSTER fed in for the agent-joined test below — must
        # match the cluster profile name registered by the server.
        - K3S_CLUSTER=k3s-srv
```

```bash
ov deploy add vm:k3s-ag1
# agent registers; ov eval k8s wait-nodes on server confirms the join.
ov eval k8s wait-nodes --cluster k3s-srv --count 2 --timeout 3m
```

## Tests

Build-scope:
- `/etc/rancher/k3s/config.yaml` exists, mode 0600.
- `/etc/systemd/system/k3s-agent.service` exists.

Deploy-scope (uses `/ov:test-k8s`):
- `k8s: wait-nodes name=${HOSTNAME}` — this node reaches Ready on the
  server.

## Related Layers
- `/ov-layers:k3s` — Base layer installing the k3s binary (required dep)
- `/ov-layers:k3s-server` — Control-plane node this agent joins
- `/ov:test-k8s` — Test verb used by the agent-joined check
