---
name: check-k8s
description: Kubernetes cluster probe verb — `charly check k8s <method>` for nodes, pods, ingress, storage class, addon health, apply/delete, and arbitrary resource GETs. Hermetic via vendored client-go; no external kubectl required.
allowed-tools: Bash, Read
---

MUST be invoked before any work involving: `charly check k8s` commands,
cluster-readiness probes from test scripts, ingress / storage class
assertions, k3s default-addon health checks, or declarative `k8s:`
steps in a candy/box `plan:` in charly.yml.

## Command surface

```
charly check k8s nodes                                                       # <name> <Ready|NotReady> per line
charly check k8s wait-nodes [--count=N] [--name=<host>] [--timeout=120s]     # block until N (or named) Ready
charly check k8s pods [--namespace=<ns>] [--label=<sel>]                     # <ns>/<name> <phase> per line
charly check k8s wait-ready --kind <K> --name <N> [--namespace=<ns>] [--timeout=120s]  # block until resource Ready
charly check k8s ingress [--namespace=<ns>]                                  # <ns>/<name> class=<c> hosts=<h> backends=<b>
charly check k8s ingressclass                                                # <name> default=<bool>
charly check k8s storageclass                                                # <name> default=<bool>
charly check k8s service [--namespace=<ns>]                                  # <ns>/<name> <type> <clusterIP> <externalIP>
charly check k8s lb-external-ip --namespace=<ns> --name=<svc> [--timeout=60s]  # print assigned external IP
charly check k8s addons [--namespace=kube-system] [--timeout=180s]           # roll-up: Traefik + ServiceLB + local-path all Ready
charly check k8s apply --file=<manifest.yaml> [--namespace=<ns>]             # apply multi-doc YAML via dynamic client
charly check k8s delete --file=<manifest.yaml> [--namespace=<ns>]            # delete resources from manifest
charly check k8s raw --resource=<plural> [--group=<g>] [--version=v1] [--name=<n>] [--namespace=<ns>]
```

## Cluster selection

Every verb accepts the same three cluster-selection flags, resolved
in this precedence:

1. `--kubeconfig <path>` — direct kubeconfig file pointer. Overrides
   everything.
2. `--cluster <name>` — a ClusterProfile name
   (`~/.config/charly/clusters/<name>.yaml` or `./clusters/<name>.yaml`).
   The profile's `kubeconfig_context:` selects the context; kubeconfig
   path defaults to `$KUBECONFIG` then `~/.kube/config`.
3. `--context <name>` — override the kubeconfig context directly.
4. Neither given → current-context of the default kubeconfig
   (matches `kubectl` with no flags).

`charly deploy add vm:k3s-srv` (or any deploy whose layers include
`k3s-server`) automatically writes a ClusterProfile named after the
deploy, so after provisioning you can do:

```bash
charly check k8s nodes --cluster k3s-srv
charly check k8s addons --cluster k3s-srv
```

## Declarative `k8s:` steps in a layer `plan:`

The verb is also callable from a candy's `plan:` steps via the `k8s:`
discriminator on a step's `Op`. Every subcommand above maps to a method
name; shared modifiers (`name:`, `namespace:`, `cluster:`, `timeout:`,
`kubeconfig:`, `k8s_kind:`, `k8s_count:`, `manifest:`, `k8s_resource:`,
`k8s_group:`, `k8s_version:`) are available. A `k8s:` step is a `check:`
step.

Example from `candy/k3s-server/charly.yml`:

```yaml
plan:
  - check: the cluster reports at least one Ready node
    k8s: wait-nodes
    cluster: "${DEPLOY_NAME}"
    k8s_count: 1
    timeout: 180s
    stdout: { contains: "Ready" }
    context: [deploy]

  # `addons` BLOCKS until Traefik + ServiceLB + local-path are all Ready, so it
  # MUST precede any ingressclass/storageclass step — those resources are
  # registered by the addon stack. Ordering matters: `ingressclass`/`storageclass`
  # are one-shot list verbs with no internal wait, and they exit 0 on an EMPTY
  # list, so a `contains` matcher run before the addons settle FAILS rather than
  # waits. Gate first, assert second.
  - check: Traefik, ServiceLB, and local-path addons are all Ready
    k8s: addons
    cluster: "${DEPLOY_NAME}"
    timeout: 240s
    context: [deploy]

  - check: Traefik is registered as the cluster's default ingress class
    k8s: ingressclass
    cluster: "${DEPLOY_NAME}"
    stdout: { contains: "traefik" }
    context: [deploy]
```

`cluster: "${DEPLOY_NAME}"` lets a candy's `context: [deploy]` step address its own
cluster generically: `${DEPLOY_NAME}` is a **runtime-only check var** resolving to
the sanitized deploy name (`:`/`.`/`/` → `-`) — the SAME identifier
`K3sPostProvision` uses for the kubeconfig context + ClusterProfile. It is
UPPERCASE because the check-var expander only recognizes uppercase names; a
lowercase `${deploy_name}` (the artifact-path token) is NOT an check var and is
rejected by `charly box validate` in k8s identifier fields.

`wait-nodes` with `name:` set matches a single specific node (used by
`k3s-agent`'s join-confirmation test). Without `name:`, it waits until
`k8s_count:` nodes are Ready.

## Method notes

- **apply / delete** — limited to the kinds in `kindToPluralResource()`
  in `k8s_cmd.go`. Static table by design; adding a new kind is a
  one-line addition, avoiding the RESTMapper discovery bloat. Documents
  without a namespace inherit `--namespace`.
- **raw** — escape hatch for any resource not covered by the named
  verbs. `charly check k8s raw --resource nodes` lists nodes;
  `charly check k8s raw --resource configmaps -n kube-system --name foo`
  prints one ConfigMap as JSON.
- **addons** — assumes the stock k3s addon stack (Traefik, ServiceLB,
  local-path-provisioner) in `kube-system`. Explicit `disable:` in a
  k3s-server layer will cause this method to fail — the failure is
  intentional since the test speaks to "default k3s stack healthy".
- **lb-external-ip** — polls `.status.loadBalancer.ingress[].ip` /
  `[].hostname` until one appears; for k3s this is ServiceLB
  (klipper-lb) advertising the host's node IP.

## Implementation

- `charly/k8s_cmd.go` — the 13 `K8s<Method>Cmd` structs + shared
  `k8sClusterFlags`. Dynamic-client via `k8s.io/client-go/dynamic` +
  `unstructured` walkers, no typed clientset (keeps binary bloat ~5-8 MB
  instead of ~15 MB with the typed clients).
- `charly/checkspec.go` — `K8s` discriminator on the step `Op` plus the shared
  resource-identity modifiers (`Name`, `Namespace`, `Label`, `Cluster`,
  `Manifest`, `K8sKind`, `K8sContext`, `Kubeconfig`, `K8sCount`,
  `K8sResource`, `K8sGroup`, `K8sVersion`); `charly/description_spec.go`
  defines the `Step` type that embeds the `Op` inline.
- `charly/checkrun_charly_verbs.go` — `k8sMethods` table + `runK8s` dispatcher
  + `posK8s*` flag builders. Matches the existing `runLibvirt` /
  `runSpice` subprocess-delegation pattern.
- `charly/k8s_config.go:162 LoadClusterProfile` — how `--cluster <name>`
  resolves to a kubeconfig + context.
