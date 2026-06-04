---
name: migrate
description: |
  MUST be invoked before any work involving: the `ov migrate` command (the single idempotent migration that brings any overthink config up to the latest schema CalVer), the CalVer schema-version stamp (`version: YYYY.DDD.HHMM`), the ordered migration chain / registry (ov/migrate_registry.go), the `LatestSchemaVersion()` load-time gate, adding a new schema cutover as a migration step, or the calver-schema int→CalVer transition.
---

# ov migrate — single-command schema migration

`ov migrate` is **one idempotent command** that brings any overthink config — however old — up to the **latest schema CalVer**, and only to the latest. There are no sub-verbs to choose between: `ov migrate` always runs the whole ordered migration chain to HEAD.

```bash
ov migrate            # migrate every reachable config to the latest schema CalVer
ov migrate --dry-run  # print every change the chain would make; touch nothing
```

The project directory is the current working directory; use the top-level `-C` / `--dir` / `OV_PROJECT_DIR` global to point elsewhere (`main()` chdir's before dispatch).

## CalVer schema versioning

The YAML schema version is a **CalVer string** — `version: YYYY.DDD.HHMM`, the same `ComputeCalVer` scheme as image tags (e.g. the current HEAD `version: 2026.144.1443`). The `calver-schema` migration step (registry HEAD) converts the legacy integer `version: 4` to this form. Every versioned file carries the stamp:

- `overthink.yml` + the per-kind siblings `image.yml` / `deploy.yml` / `vm.yml` / `pod.yml` / `k8s.yml` / `local.yml` (and `eval.yml` when present)
- the per-host `~/.config/ov/deploy.yml`

`ov/version.go` provides `ParseCalVer(string) (CalVer, bool)` and `CalVer.Less` (generation-only `ComputeCalVer` is unchanged). `LatestSchemaVersion()` (in `ov/migrate_registry.go`) is the curated **HEAD** value — the constant every file is stamped to and the value the load-time gate requires.

### Per-push release git tags (decoupled from `version:`)

Every push of an ov-project repo (one with an `overthink.yml`) carries a **fresh** annotated release git tag `v<YYYY.DDD.HHMM>` computed from the current UTC push time — ONE per push, so a repo accumulates MULTIPLE `v…` tags over time. This is **decoupled** from the `overthink.yml` `version:` field: `version:` is the SCHEMA version (bumped only when `ov migrate` raises it to a new `LatestSchemaVersion()`), whereas the tag marks the push moment. Tag EVERY push — including one at an unchanged `version:` (a content change, submodule extraction, image drop). Tags are **immutable**: only ever ADD new ones, never move or force-push an existing tag (so the load-time gate never sees a "newer than supported" CalVer from a re-tag). The day-of-year is NOT zero-padded (`v2026.64.1937`, `v2026.142.1640`); compute with `v$(date -u +%Y).$((10#$(date -u +%j))).$(date -u +%H%M)`, never bare `+%Y.%j.%H%M`. Each repo (the main repo and every `image/<distro>` submodule) is tagged independently at its own push time; push submodule(s) first, then the superproject, then the tag on the pushed HEAD. Repos without an `overthink.yml` (`plugins`, `pkg/arch`) are out of scope. See CLAUDE.md "Post-Execution Policies".

## The migration chain (registry)

Every hard-cutover schema change the project has ever shipped is **one `MigrationStep`** in the ordered registry `migrationSteps()` (`ov/migrate_registry.go`), stamped with the CalVer of the date it landed and listed **chronologically** — the order the cutovers were authored in, which is the only correct replay order for an arbitrarily-old config. `ov migrate` runs the whole chain; each step's own idempotency guard makes running it whole safe (already-current files are no-ops).

| CalVer | Step | What it does |
|---|---|---|
| 2026.112.522 | unified | legacy `image.yml`+`build.yml`+flat `layer.yml` → `overthink.yml` (kind-keyed wrappers + the then-current composition key, since renamed to `import:` by the 2026.143.843 step); raw-INI `service:` / `system_services:` → structured `service:` list |
| 2026.114.1558 | schema-v4 | v3 → v4: flatten `deployments.images.*` → `deployment.*`, plural→singular root keys, `children:`→`nested:`, `target: container`→`pod` etc. |
| 2026.114.2207 | description | scaffold a Gherkin `description:` block from legacy `info:`/`status:` |
| 2026.123.114 | target-local | `kind: host`→`kind: local`, `host.yml`→`local.yml`, `target: host`→`target: local` |
| 2026.123.1351 | calamares | layer.yml: `depends:`→`requires:`, collapse `rpm:`/`deb:`/`pac:`/`aur:` + per-distro tag sections into `packages:` + `distros:` map |
| 2026.124.1942 | shell-schema | legacy shell-rc heredoc `cmd:` tasks → structured `shell:` schema |
| 2026.125.702 | ov-cachyos | collapse `qc` → `cachyos-dx` → `ov-cachyos` deployment + kind:local template names |
| 2026.125.1107 | local-images | kind:local `images:` field → dated comment fence (deploy-fetch narrowing) |
| 2026.125.2355 | kind-files | split inline `image:`/`vm:` into sibling files, stub `pod.yml`/`k8s.yml`, `kind: deployment`→`kind: deploy`, root `deployment:`→`deploy:` |
| 2026.128.255 | local-deploy | `~/.config/ov/deploy.yml`: `images:`→`deploy:`, `bind_mounts:`→`volumes:`, `workspace:` scalar → bind volume |
| 2026.128.306 | quadlets | regenerate encrypted-volume quadlets missing the `ExecStartPre=ov config mount` auto-mount hook |
| 2026.130.1530 | field-singular | singularize every plural overthink-NATIVE authoring key project-wide (`layers:`→`layer:`, `ports:`→`port:`, `platforms:`→`platform:`, `env_provide:`→`env_provide:`, …; `builds:`→`produce:`). EXTERNAL-schema keys are kept PLURAL — fields rendered to cloud-init / Kubernetes / libvirt config use those schemas' own plural keys (`users:`, `labels:`, `resources:`, `<devices>`, `<topology sockets= cores=>`), so authoring them in the same plural spelling keeps a 1:1 mapping. The kept-plural set = keys living in `cloud_init_types.go` / `k8s_spec.go` / `k8s_config.go` / `libvirt_yaml.go`. Verb-collision carve-outs: `groups`/`addrs` |
| 2026.131.857 | marimo-rename | `marimo-ml` image + `marimo-ml-pod` deploy → `versa` (cross-kind name reuse) |
| 2026.132.1009 | require-image | inject the required `image:` field on every `target: pod` deploy entry |
| 2026.132.2311 | tailscale-secrets | rename flat `TS_AUTHKEY` → per-tailnet `TS_AUTHKEY_<TAILNET>` (non-interactive: auto-detect from a running sidecar or warn) |
| 2026.141.1326 | drop-kdbx | strip residual `secret_backend: kdbx` + `secrets_kdbx_*` keys from `~/.config/ov/config.yml` |
| 2026.141.1559 | arch-rename | rename the `archlinux` distro tag + `archlinux` / `archlinux-builder` / `archlinux-pacstrap*` image identifiers to `arch` / `arch-builder` / `arch-pacstrap*` project-wide (overthink.yml + siblings + build.yml + base.yml + every layer.yml + per-host deploy.yml). EXTERNAL Arch strings stay verbatim: ANY external registry ref whose path contains `archlinux` is protected by SHAPE (`archRenameExternalRefRe` — covers `quay.io/archlinux/archlinux`, `docker.io/library/archlinux`, `ghcr.io/<ns>/archlinux-*`, `host:port/.../archlinux`), plus `pacman-key --populate archlinux`, `archlinux.org` mirror URLs, `archlinux-keyring`. Idempotent (a renamed config has no non-external `archlinux` left) |
| 2026.143.843 | import-namespace | the **2026-05 import-namespace cutover**: rename the deleted top-level `include:` composition key to `import:` in `overthink.yml` + every per-kind sibling + `build.yml` / `base.yml` (and the legacy `arch-base.yml` / `fedora-base.yml` / `cachyos-base.yml` if present), preserving the value list verbatim (a flat include → a flat import, same root-namespace-merge). Comment-preserving (yaml.v3 node API); idempotent (a config already on `import:` is a no-op). Repo-specific reshaping — combining `arch-base.yml` + `fedora-base.yml` into `base.yml`, mounting the `cachyos` namespace, the deploy→eval bed move — is hand-authored in the cutover (recorded in `CHANGELOG.md`), NOT done by this step; a third-party config that only flat-includes its own files migrates cleanly with no behavior change |
| 2026.144.1442 | entity-version | the **2026-05 per-kind versioning cutover**: backfill the per-entity `version:` field that became load-bearing. Injects `version: <HEAD>` on EVERY `layers/<name>/layer.yml` (the layer kind now requires it) + every BARE-BASE image entry (no `layer:` field AND an external registry `base:`, e.g. `arch`/`fedora`). Layered + internal-base images are left UNVERSIONED (they derive the highest layer version). Skips the `image/` submodules (each migrates in its own repo) + `testdata`. Comment-preserving (yaml.v3 node API); idempotent; never touches the document-root schema stamp. `TouchesHost: false` so remote-cache auto-migration backfills fetched remote layers (the runtime then hard-errors on an unversioned fetched layer instead of carrying a fallback). See CHANGELOG.md |
| **2026.144.1443** | **calver-schema** (HEAD) | **the universal stamper**: rewrite the top-level `version:` line of every versioned file to the HEAD CalVer (line-oriented, comment-preserving). This is the integer→CalVer transition (`version: 4` → the HEAD CalVer). **Always stays last** so `LatestSchemaVersion()` tracks it. |

Step `Name`s are for `--dry-run` / progress output only — they are no longer CLI sub-verbs. Steps that mutate per-host state (`~/.config/ov`, quadlets, `.secrets`) are flagged `TouchesHost`; see "Remote-cache auto-migration" below.

## How it runs

`RunMigrations(ctx)` walks `migrationSteps()` in order, calling each `Apply(ctx)`. Each step reports whether it changed anything; nothing-changed across the whole chain prints `nothing to migrate (already at schema <HEAD>)`. Per-step backups follow the established `<file>.bak.<unix-ts>` convention (field-singular, local-deploy, require-image, drop-kdbx, and the calver-schema stamp write them). `MigrateContext` (built by `NewMigrateContext`) carries the project dir, the per-host paths, the quadlet dir, the `.secrets` path, and `DryRun`.

### Remote-cache auto-migration (project-only)

`ov/refs.go` auto-runs `RunProjectMigrations(ctx)` on a freshly-cloned remote-repo cache so external repos pull through at the latest schema. `RunProjectMigrations` skips every `TouchesHost` step and leaves `HostDeployPath` empty, so a remote fetch **never mutates the user's per-host state** — even the calver-schema stamp touches only the cache's project files.

## Load-time gate

`LoadUnified` (`ov/unified.go`) parses the merged `version:` and rejects anything older than HEAD:

```
overthink.yml: schema 2026.144.1443 is required (found "4"). Run: ov migrate
```

A non-CalVer value (the legacy integer `4`, empty, or garbage) parses as "older than every real CalVer", so a pre-CalVer config flows into the chain with no special case. Residual-key checks (e.g. `kind: deployment`, `target: host`, `secret_backend: kdbx`) remain as defense-in-depth, but **every** remediation hint now points uniformly at bare `ov migrate`.

## Adding a future cutover

1. Write the transform as an idempotent `Migrate*` function (its own file, like the existing migrators).
2. Append ONE `MigrationStep` to `migrationSteps()` with a CalVer **strictly greater** than the current HEAD. Keep the `calver-schema` stamp **last**.
3. Bump `latestSchemaVersion` to the new step's CalVer (the two are asserted equal by `TestRegistryHeadMatchesLatest`).
4. Update the HEAD-CalVer fixtures (the LoadUnified test fixtures + `testdata/*.yml` carry the stamp) and the repo's own versioned YAML.

The operator command never changes — it stays `ov migrate`.

### Standing rule: a schema/format change bumps `version:` AND mints a git tag

Any change to the YAML schema or composition format (a key rename, a deleted key, a new key shape — e.g. the 2026-05 `include:` → `import:` cutover) is a hard-cutover that MUST:

1. **Bump the `version:` CalVer** by appending a `MigrationStep` that raises `LatestSchemaVersion()` (the `calver-schema` stamp stays last). The load-time gate then rejects any not-yet-migrated config with a `Run: ov migrate` hint, so every reader sees the new format. Bumping `version:` WITHOUT a step that raises HEAD is forbidden — and `version:` is NEVER set above `LatestSchemaVersion()` (newer configs hard-fail at load).
2. **Mint a fresh per-push git tag** on the landing push — `v<YYYY.DDD.HHMM>` from the push moment (see "Per-push release git tags" above). The tag and the `version:` bump are decoupled: the tag marks the push, `version:` marks the schema. A schema cutover happens to do BOTH at once, but a content-only push (no schema change) still mints a tag at an unchanged `version:`.

## Idempotency

Running `ov migrate` twice is a no-op (`TestMigrateCalverSchema_StampsAndIdempotent`, plus each migrator's own idempotency tests). The registry invariants (HEAD == `LatestSchemaVersion()`, strictly-ascending order, unique names) are enforced by `ov/migrate_registry_test.go`.

## See Also

- `/ov-image:layer` — the `layer:` kind-keyed schema the chain produces
- `/ov-image:image` — `image:` entries + `ov image build/validate/inspect`
- `/ov-core:deploy` — `deploy:` entries the chain migrates (require-image, ov-cachyos)
- `/ov-local:local-spec` — `kind: local` templates (target-local, local-images)
- `/ov-build:secrets`, `/ov-build:settings` — credential schema (drop-kdbx, tailscale-secrets)
- `/ov-internals:go` — loader internals (`LoadUnified`, `ParseCalVer`, `RunMigrations`, `migrationSteps`)
- `/ov-internals:cutover-policy` — why hard-cutover + a single idempotent `ov migrate` is the required shape
