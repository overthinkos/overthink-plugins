---
name: pod
description: Schema reference for `kind: pod` and `kind: bundle` entities — charly.yml entry shape, tree-position nesting, sidecars, pod networking. For verb-level operations see /charly-core:deploy.
---

# `kind: pod` and `kind: bundle` — Schema Reference

This skill is a thin schema pointer. For runtime verbs (`charly bundle add`, `charly bundle del`, `charly update`), see `/charly-core:deploy`.

## What lives in `kind: pod` / `kind: bundle`

A `pod` entity declares a co-scheduled set of containers and the volumes / network / sidecars they share. A `bundle` entity (the migrated `deploy:` kind) is the **bind site** — it picks an image (or image+layers) through a single cross-ref scalar (`box:` deploys as a pod, the default substrate; `vm:` / `k8s:` / `local:` / `android:` target the named substrate), and stores the runtime knobs (encrypted volumes, tunnels, env, ports).

Schema sources (read these for the canonical truth):

- `charly/deploy.go` — `BundleConfig` + `BundleNode` Go types, the `bundle` entry shape, target discriminator.
- `charly/pod_spec.go` — `PodSpec` Go type, `kind: pod` shape.
- `/charly-core:deploy` — the verb-level skill covering `charly bundle add` / `charly bundle del` / `charly update`.

## Nesting & membership (tree position)

Nesting is expressed by **tree position**, not a `nested:` field: a resource node placed UNDER another resource node deploys INTO it (the migrated `nested:` — e.g. a `pod → android` tree), and the parent and its nested entries share the pod and the tunnel; each nested entry is a separate quadlet/process inside the same pod namespace. A resource node placed directly under a `bundle` is instead a sibling member (the migrated `peer:`). There are no authored `nested:` / `peer:` / `target:` / `on:` fields — membership is read from the tree.

## Sidecars

`sidecar:` declares co-running containers with their own env-var routing (`env_accept` / `env_require`). See `/charly-automation:sidecar` for the topic skill.

## Cross-references

- Verb-level: `/charly-core:deploy`, `/charly-core:charly-update`, `/charly-core:remove`.
- Sibling kinds: `/charly-image:image`, `/charly-vm:vm`, `/charly-kubernetes:kubernetes`, `/charly-local:local-spec`.
- Topics: `/charly-automation:sidecar`, `/charly-automation:enc`.
