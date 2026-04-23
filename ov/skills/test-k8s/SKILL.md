---
name: ov:test-k8s
description: Kubernetes cluster probe verb — `ov test k8s <method>` for nodes, pods, ingress, storage class, addon health, apply/delete, and arbitrary resource GETs. Hermetic via vendored client-go; no external kubectl required.
allowed-tools: Bash, Read
---

MUST be invoked before any work involving: `ov test k8s` commands,
cluster-readiness probes from test scripts, ingress / storage class
assertions, k3s default-addon health checks, or declarative `k8s:`
checks on `tests:` blocks in layer.yml.

## Command surface

```
ov test k8s nodes                                                       # <name> <Ready|NotReady> per line
ov test k8s wait-nodes [--count=N] [--name=<host>] [--timeout=120s]     # block until N (or named) Ready
ov test k8s pods [--namespace=<ns>] [--label=<sel>]                     # <ns>/<name> <phase> per line
ov test k8s wait-ready --kind <K> --name <N> [--namespace=<ns>] [--timeout=120s]  # block until resource Ready
ov test k8s ingress [--namespace=<ns>]                                  # <ns>/<name> class=<c> hosts=<h> backends=<b>
ov test k8s ingressclass                                                # <name> default=<bool>
ov test k8s storageclass                                                # <name> default=<bool>
ov test k8s service [--namespace=<ns>]                                  # <ns>/<name> <type> <clusterIP> <externalIP>
ov test k8s lb-external-ip --namespace=<ns> --name=<svc> [--timeout=60s]  # print assigned external IP
ov test k8s addons [--namespace=kube-system] [--timeout=180s]           # roll-up: Traefik + ServiceLB + local-path all Ready
ov test k8s apply --file=<manifest.yaml> [--namespace=<ns>]             # apply multi-doc YAML via dynamic client
ov test k8s delete --file=<manifest.yaml> [--namespace=<ns>]            # delete resources from manifest
ov test k8s raw --resource=<plural> [--group=<g>] [--version=v1] [--name=<n>] [--namespace=<ns>]
```

## Cluster selection

Every verb accepts the same three cluster-selection flags, resolved
in this precedence:

1. `--kubeconfig <path>` — direct kubeconfig file pointer. Overrides
   everything.
2. `--cluster <name>` — a ClusterProfile name
   (`~/.config/ov/clusters/<name>.yaml` or `./clusters/<name>.yaml`).
   The profile's `kubeconfig_context:` selects the context; kubeconfig
   path defaults to `$KUBECONFIG` then `~/.kube/config`.
3. `--context <name>` — override the kubeconfig context directly.
4. Neither given → current-context of the default kubeconfig
   (matches `kubectl` with no flags).

`ov deploy add vm:k3s-srv` (or any deploy whose layers include
`k3s-server`) automatically writes a ClusterProfile named after the
deploy, so after provisioning you can do:

```bash
ov test k8s nodes --cluster k3s-srv
ov test k8s addons --cluster k3s-srv
```

## Declarative `k8s:` checks on layer tests

The verb is also callable from a layer's `tests:` block via the `k8s:`
discriminator field on `Check`. Every subcommand above maps to a method
name; shared modifiers (`name:`, `namespace:`, `cluster:`, `timeout:`,
`kubeconfig:`, `k8s_kind:`, `k8s_count:`, `manifest:`, `k8s_resource:`,
`k8s_group:`, `k8s_version:`) are available.

Example from `layers/k3s-server/layer.yml`:

```yaml
tests:
  - id: cluster-nodes-ready
    scope: deploy
    k8s: wait-nodes
    cluster: "${deploy_name}"
    k8s_count: 1
    timeout: 180s
    stdout: { contains: "Ready" }

  - id: traefik-ingressclass
    scope: deploy
    k8s: ingressclass
    cluster: "${deploy_name}"
    stdout: { contains: "traefik" }

  - id: addons-healthy
    scope: deploy
    k8s: addons
    cluster: "${deploy_name}"
    timeout: 240s
```

`wait-nodes` with `name:` set matches a single specific node (used by
`k3s-agent`'s join-confirmation test). Without `name:`, it waits until
`k8s_count:` nodes are Ready.

## Method notes

- **apply / delete** — limited to the kinds in `kindToPluralResource()`
  in `k8s_cmd.go`. Static table by design; adding a new kind is a
  one-line addition, avoiding the RESTMapper discovery bloat. Documents
  without a namespace inherit `--namespace`.
- **raw** — escape hatch for any resource not covered by the named
  verbs. `ov test k8s raw --resource nodes` lists nodes;
  `ov test k8s raw --resource configmaps -n kube-system --name foo`
  prints one ConfigMap as JSON.
- **addons** — assumes the stock k3s addon stack (Traefik, ServiceLB,
  local-path-provisioner) in `kube-system`. Explicit `disable:` in a
  k3s-server layer will cause this method to fail — the failure is
  intentional since the test speaks to "default k3s stack healthy".
- **lb-external-ip** — polls `.status.loadBalancer.ingress[].ip` /
  `[].hostname` until one appears; for k3s this is ServiceLB
  (klipper-lb) advertising the host's node IP.

## Implementation

- `ov/k8s_cmd.go` — the 13 `K8s<Method>Cmd` structs + shared
  `k8sClusterFlags`. Dynamic-client via `k8s.io/client-go/dynamic` +
  `unstructured` walkers, no typed clientset (keeps binary bloat ~5-8 MB
  instead of ~15 MB with the typed clients).
- `ov/testspec.go:55+` — `K8s` discriminator on `Check` plus the shared
  resource-identity modifiers (`Name`, `Namespace`, `Label`, `Cluster`,
  `Manifest`, `K8sKind`, `K8sContext`, `Kubeconfig`, `K8sCount`,
  `K8sResource`, `K8sGroup`, `K8sVersion`).
- `ov/testrun_ov_verbs.go` — `k8sMethods` table + `runK8s` dispatcher
  + `posK8s*` flag builders. Matches the existing `runLibvirt` /
  `runSpice` subprocess-delegation pattern.
- `ov/k8s_config.go:162 LoadClusterProfile` — how `--cluster <name>`
  resolves to a kubeconfig + context.
