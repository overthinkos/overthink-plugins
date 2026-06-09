---
name: pod
description: Schema reference for `kind: pod` and `kind: deploy` entities — deploy.yml entry shape, nested deploys, sidecars, pod networking. For verb-level operations see /charly-core:deploy.
---

# `kind: pod` and `kind: deploy` — Schema Reference

This skill is a thin schema pointer. For runtime verbs (`charly deploy add`, `charly deploy del`, `charly update`), see `/charly-core:deploy`.

## What lives in `kind: pod` / `kind: deploy`

A `pod` entity declares a co-scheduled set of containers and the volumes / network / sidecars they share. A `deploy` entity is the **bind site** — it picks an image (or image+layers), maps it onto a target (`pod`, `local`, `vm`, `k8s`), and stores the runtime knobs (encrypted volumes, tunnels, env, ports).

Schema sources (read these for the canonical truth):

- `charly/deploy_spec.go` — `DeploySpec` Go type, `kind: deploy` shape, target discriminator.
- `charly/pod_spec.go` — `PodSpec` Go type, `kind: pod` shape.
- `/charly-core:deploy` — the verb-level skill covering `charly deploy add` / `charly deploy del` / `charly update`.

## Nested deploys

`deploy` entries can nest via `nested:`. A parent `deploy` and its nested entries share the pod and the tunnel; each nested entry is a separate quadlet/process inside the same pod namespace.

## Sidecars

`sidecar:` declares co-running containers with their own env-var routing (`env_accept` / `env_require`). See `/charly-automation:sidecar` for the topic skill.

## Cross-references

- Verb-level: `/charly-core:deploy`, `/charly-core:charly-update`, `/charly-core:remove`.
- Sibling kinds: `/charly-image:image`, `/charly-vm:vm`, `/charly-kubernetes:kubernetes`, `/charly-local:local-spec`.
- Topics: `/charly-automation:sidecar`, `/charly-automation:enc`.
