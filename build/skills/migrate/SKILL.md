---
name: migrate
description: |
  MUST be invoked before any work involving: the `charly migrate` command (the single idempotent migration that brings any opencharly config up to the latest schema CalVer), the CalVer schema-version stamp (`version: YYYY.DDD.HHMM`), the ordered migration chain / registry (charly/migrate_registry.go), the `LatestSchemaVersion()` load-time gate, adding a new schema cutover as a migration step, or the calver-schema int→CalVer transition.
---

# charly migrate — single-command schema migration

`charly migrate` is **one idempotent command** that brings any opencharly config — however old — up to the **latest schema CalVer**, and only to the latest. There are no sub-verbs to choose between: `charly migrate` always runs the whole ordered migration chain to HEAD.

```bash
charly migrate            # migrate every reachable config to the latest schema CalVer
charly migrate --dry-run  # print every change the chain would make; touch nothing
```

The project directory is the current working directory; use the top-level `-C` / `--dir` / `CHARLY_PROJECT_DIR` global to point elsewhere (`main()` chdir's before dispatch).

## CalVer schema versioning

The YAML schema version is a **CalVer string** — `version: YYYY.DDD.HHMM`, the same `ComputeCalVer` scheme as image tags (e.g. the current HEAD `version: 2026.165.1048`). It is CANONICAL fixed-width: a 4-digit year, a 3-digit zero-padded day-of-year, and a 4-digit zero-padded HHMM, so a plain alphanumeric sort of CalVer strings is chronological. `ParseCalVer` is EXTREMELY STRICT — it accepts ONLY that exact form (no `version: 4`, no non-padded `2026.45.830`), and a non-canonical value is "not a CalVer", which the load gate treats as older-than-HEAD so it flows into `charly migrate` and is re-stamped canonical. The `calver-schema` migration step (registry HEAD) converts the legacy integer `version: 4` to this form. Every versioned file carries the stamp:

- every project `charly.yml` (splitting into per-kind `vm.yml` / `local.yml` / … siblings is an optional `import:` convenience a project MAY use — never the default; see `/charly-image:image`)
- the per-host `~/.config/charly/charly.yml`

`charly/version.go` provides `ParseCalVer(string) (CalVer, bool)` and `CalVer.Less` (generation-only `ComputeCalVer` is unchanged). `LatestSchemaVersion()` (in `charly/migrate_registry.go`) is the curated **HEAD** value — the constant every file is stamped to and the value the load-time gate requires.

### Per-push release git tags (decoupled from `version:`)

Every push of an charly-project repo (one with an `charly.yml`) carries a **fresh** annotated release git tag `v<YYYY.DDD.HHMM>` computed from the current UTC push time — ONE per push, so a repo accumulates MULTIPLE `v…` tags over time. This is **decoupled** from the `charly.yml` `version:` field: `version:` is the SCHEMA version (bumped only when `charly migrate` raises it to a new `LatestSchemaVersion()`), whereas the tag marks the push moment. Tag EVERY push — including one at an unchanged `version:` (a content change, submodule extraction, image drop). Tags are **immutable**: only ever ADD new ones, never move or force-push an existing tag (so the load-time gate never sees a "newer than supported" CalVer from a re-tag). Every component is fixed-width zero-padded — 4-digit year, 3-digit day-of-year, 4-digit HHMM (`v2026.064.1937`, `v2026.142.1640`) — so tags sort chronologically under a plain alphanumeric sort; compute with `v$(date -u +%Y.%j.%H%M)` (`%j` and `%H%M` are already zero-padded). Each repo (the main repo and every `box/<distro>` submodule) is tagged independently at its own push time; push submodule(s) first, then the superproject, then the tag on the pushed HEAD. Repos without an `charly.yml` (`plugins`, `pkg/arch`) are out of scope. See CLAUDE.md "Post-Execution Policies".

## The migration chain (registry)

Every hard-cutover schema change the project has ever shipped is **one `MigrationStep`** in the ordered registry `migrationSteps()` (`charly/migrate_registry.go`), stamped with the CalVer of the date it landed and listed **chronologically** — the order the cutovers were authored in, which is the only correct replay order for an arbitrarily-old config. `charly migrate` runs the whole chain; each step's own idempotency guard makes running it whole safe (already-current files are no-ops).

| CalVer | Step | What it does |
|---|---|---|
| 2026.112.522 | unified | legacy `box.yml`+`build.yml`+flat `candy.yml` → `charly.yml` (kind-keyed wrappers + the then-current composition key, since renamed to `import:` by the 2026.143.843 step); raw-INI `service:` / `system_services:` → structured `service:` list |
| 2026.114.1558 | schema-v4 | v3 → v4: flatten `deployments.images.*` → `deployment.*`, plural→singular root keys, `children:`→`nested:`, `target: container`→`pod` etc. |
| 2026.114.2207 | description | scaffold a Gherkin `description:` block from legacy `info:`/`status:` |
| 2026.123.114 | target-local | `kind: host`→`kind: local`, `host.yml`→`local.yml`, `target: host`→`target: local` |
| 2026.124.1942 | shell-schema | legacy shell-rc heredoc `cmd:` tasks → structured `shell:` schema |
| 2026.125.702 | charly-cachyos | collapse `qc` → `cachyos-dx` → `charly-cachyos` deployment + kind:local template names |
| 2026.125.1107 | local-images | kind:local `images:` field → dated comment fence (deploy-fetch narrowing) |
| 2026.125.2355 | kind-files | split inline `image:`/`vm:` into sibling files, stub `pod.yml`/`k8s.yml`, `kind: deployment`→`kind: deploy`, root `deployment:`→`deploy:` |
| 2026.128.255 | local-deploy | `~/.config/charly/deploy.yml`: `images:`→`deploy:`, `bind_mounts:`→`volumes:`, `workspace:` scalar → bind volume |
| 2026.128.306 | quadlets | regenerate encrypted-volume quadlets missing the `ExecStartPre=charly config mount` auto-mount hook |
| 2026.130.1530 | field-singular | singularize every plural opencharly-NATIVE authoring key project-wide (`layers:`→`layer:`, `ports:`→`port:`, `platforms:`→`platform:`, `env_provide:`→`env_provide:`, …; `builds:`→`produce:`). EXTERNAL-schema keys are kept PLURAL — fields rendered to cloud-init / Kubernetes / libvirt config use those schemas' own plural keys (`users:`, `labels:`, `resources:`, `<devices>`, `<topology sockets= cores=>`), so authoring them in the same plural spelling keeps a 1:1 mapping. The kept-plural set = keys living in `cloud_init_types.go` / `k8s_spec.go` / `k8s_config.go` / `libvirt_yaml.go`. Verb-collision carve-outs: `groups`/`addrs` |
| 2026.131.857 | marimo-rename | `marimo-ml` image + `marimo-ml-pod` deploy → `versa` (cross-kind name reuse) |
| 2026.132.1009 | require-image | inject the required `image:` field on every `target: pod` deploy entry |
| 2026.132.2311 | tailscale-secrets | rename flat `TS_AUTHKEY` → per-tailnet `TS_AUTHKEY_<TAILNET>` (non-interactive: auto-detect from a running sidecar or warn) |
| 2026.141.1326 | drop-kdbx | strip residual `secret_backend: kdbx` + `secrets_kdbx_*` keys from `~/.config/charly/config.yml` |
| 2026.141.1559 | arch-rename | rename the `archlinux` distro tag + `archlinux` / `archlinux-builder` / `archlinux-pacstrap*` image identifiers to `arch` / `arch-builder` / `arch-pacstrap*` project-wide (charly.yml + siblings + build.yml + base.yml + every candy.yml + per-host deploy.yml). EXTERNAL Arch strings stay verbatim: ANY external registry ref whose path contains `archlinux` is protected by SHAPE (`archRenameExternalRefRe` — covers `quay.io/archlinux/archlinux`, `docker.io/library/archlinux`, `ghcr.io/<ns>/archlinux-*`, `host:port/.../archlinux`), plus `pacman-key --populate archlinux`, `archlinux.org` mirror URLs, `archlinux-keyring`. Idempotent (a renamed config has no non-external `archlinux` left) |
| 2026.143.843 | import-namespace | the **2026-05 import-namespace cutover**: rename the deleted top-level `include:` composition key to `import:` in `charly.yml` + every per-kind sibling + `build.yml` / `base.yml` (and the legacy `arch-base.yml` / `fedora-base.yml` / `cachyos-base.yml` if present), preserving the value list verbatim (a flat include → a flat import, same root-namespace-merge). Comment-preserving (yaml.v3 node API); idempotent (a config already on `import:` is a no-op). Repo-specific reshaping — combining `arch-base.yml` + `fedora-base.yml` into `base.yml`, mounting the `cachyos` namespace, the deploy→check bed move — is hand-authored in the cutover (recorded in `CHANGELOG/`), NOT done by this step; a third-party config that only flat-includes its own files migrates cleanly with no behavior change |
| 2026.144.1442 | entity-version | the **2026-05 per-kind versioning cutover**: backfill the per-entity `version:` field that became load-bearing. Injects `version: <HEAD>` on EVERY `candy/<name>/candy.yml` (the layer kind now requires it) + every BARE-BASE image entry (no `layer:` field AND an external registry `base:`, e.g. `arch`/`fedora`). Layered + internal-base images are left UNVERSIONED (they derive the highest layer version). Skips any NESTED git submodule (the `box/<distro>` distro submodules, `plugins`, `pkg/*` — each migrates in its OWN repo / via remote-cache auto-migration), detected generically by a nested `.git` so the skip is independent of where a submodule is mounted (`isNestedGitRepo`), + `testdata`. Comment-preserving (yaml.v3 node API); idempotent; never touches the document-root schema stamp. `TouchesHost: false` so remote-cache auto-migration backfills fetched remote layers (the runtime then hard-errors on an unversioned fetched layer instead of carrying a fallback). See CHANGELOG/ |
| 2026.155.1800 | singular-label | singularize plural `org.overthinkos.<seg>` OCI-label NAME references (build.yml init `label_key`, a forked `oci_label:` override, an check `command:` that inspects a label); prefix-anchored on `org.overthinkos.` so a bare authoring key is never touched |
| 2026.156.556 | candy-box-rename | the 2026-06 candy/box rebrand: schema kinds `layer:`→`candy:` + `image:`→`box:` at every depth, the per-kind filenames (`image.yml`→`box.yml`, `layer.yml`→`candy.yml`) + the `layers/`→`candy/` directory; rewrites `import:`/`discover:` path references |
| 2026.156.1040 | discover-flatten | kind-keyed `discover: {candy: [...]}` → FLAT generic scan-spec list `discover: [{path, recursive, manifest}]` (files are generic kind-containers routed by shape; no per-kind filename baked into the loader) |
| 2026.156.1530 | peer-field | additive `peer:` map (sibling companion deployments) on `kind: check` / `kind: deploy`; transforms nothing — raises HEAD so an older binary REJECTS a `peer:`-using config (with a `Run: charly migrate` hint) instead of silently dropping the key |
| 2026.157.0310 | localpkg-map | layer `localpkg:` scalar → per-format map `{pac: <dir>}` (one `charly` layer carries a native-package SOURCE per distro format); the loader hard-rejects the legacy scalar form |
| 2026.159.0002 | charly-rebrand | the ov→charly / overthink→opencharly rebrand: `overthink.yml`→`charly.yml`, `@github…/candy/ov[-mcp]` ref paths, `org.overthinkos.*` labels→`ai.opencharly.*`, the import alias `ov`→`charly`, and host-gated relocation of the per-host state dirs (`~/.config/ov`→`charly`, etc.) with `OV_*`→`CH_*` env-key rewrites. The `overthinkos` GitHub org + ghcr registry + repo names are KEPT |
| 2026.159.1911 | charly-cutover4 | finish the rebrand: `CH_*`→`CHARLY_*` env, credential service prefix `ov/`→`charly/` (incl. OS-keyring re-key), image names `arch-charly`→`charly-arch` / `fedora-charly`→`charly-fedora`, and the fish shell-init `overthink.fish`→`opencharly.fish` + markers. Host transforms gated on `ctx.HostDeployPath` |
| 2026.160.1300 | single-filename | charly.yml becomes the ONE filename for box + candy: boxes split out of box.yml/base.yml (and an inline `box:` map, e.g. bootc) into discovered `box/<name>/charly.yml`; candy manifests rename `candy.yml`→`charly.yml`; the per-kind files (`vm`/`pod`/`k8s`/`check`/`local`/`android`) fold into `charly.yml`'s root; the `build.yml` import is dropped (the distro/builder/init/resource vocabulary is now EMBEDDED in the binary at `charly/charly.yml`); `discover:` is rewritten to scan `box/` + `candy/`. TouchesHost false — runs under remote-cache auto-migration so a fetched remote's candy manifests rename too |
| 2026.161.1300 | recipe-section-values | finish the candy/box rebrand's check-harness recipe `from[i].kind:`/`scope:` VALUES (layer→candy, image→box), scoped to `from:` sequence items; the check label wire keys were already candy/box. The load gate hard-rejects an un-migrated recipe (`invalid kind "layer" (one of: candy, box, pod, vm)`) |
| 2026.161.1501 | init-candy-keys | finish the candy/box rebrand's INIT-SYSTEM vocabulary — `layer_field:`→`candy_field:`, `layer_file:`→`candy_file:`, `depends_layer:`→`depends_candy:` inside `init:` system defs (`build.yml` / `charly.yml`). The Go `InitDef` now reads `candy_*`; an un-migrated config silently loses the keys. Scoped to the `init:` subtree; TouchesHost false (remote-cache auto-migration applies it to fetched init overrides) |
| 2026.161.1554 | host-charly-yml | rename the per-host deploy overlay `~/.config/charly/deploy.yml` → `charly.yml` so it loads through the SAME unified loader as every project `charly.yml` (Cutover E). Prepends a `version:` line (per-host configs predate per-file versioning, so they have none — and `stampVersionField` only rewrites an existing line) so the calver-schema stamp + the loader's gate accept it. TouchesHost; retargets `ctx.HostDeployPath`; idempotent |
| 2026.161.1649 | ledger-candy-keys | rename the install-ledger json keys `layer`→`candy` / `add_layer`→`add_candy` and stamp each record with the ledger format version (Cutover F). The install ledger (`~/.config/opencharly/installed/{deploys,layers}/*.json`) is per-host deploy STATE the unified loader never parses, so it carries its OWN fixed format version — the `ledgerSchemaVersion` constant (`2026.161.1649`, DECOUPLED from project HEAD so a future non-ledger cutover never invalidates a migrated ledger). The new read path (`ReadDeployRecord`/`ReadCandyRecord`) hard-rejects a record lacking `schema_version` (a pre-cutover record whose `json:"layer"` key would silently unmarshal to an empty `Candy`) with a `charly migrate` hint; this step converts existing ledgers so they pass. TouchesHost (walks `ctx.LedgerRoot/{deploys,layers}`); raw-JSON rewrite (no record schema — key order is irrelevant for the machine-managed ledger); idempotent (a record already carrying `schema_version` is a no-op) |
| 2026.161.2302 | drop-box-port | the candy-port-inheritance cutover: the box-level `port:` field is RETIRED (published ports are inherited from the candy chain — `CollectBoxPorts`) and host mappings are auto-allocated on `127.0.0.1` at deploy, so the `port: [auto]` sentinel is retired too (absence of pins IS auto). Removes box-level `port:` + `defaults.port` + the auto sentinel from deploy/check/pod/k8s entries (incl. nested:/peer: children); explicit deploy port PINS (host:container) are preserved. The loader hard-rejects a residual box `port:` (`rejectLegacyBoxPort`) with a `charly migrate` hint. Node-API, comment-preserving, idempotent. TouchesHost false |
| 2026.163.0927 | agent-kind-rename | the reusable agent-CLI catalog kind `kind: ai` → `kind: agent`: the catalog map key `ai:` + the kind:score eligible-agent selector `ai:` → `agent:`, and the standalone-doc `kind: ai` value → `agent`. The loader now reads `AgentConfig`/`agent:`; the residual-key gate (`rejectLegacyAgentCatalog`) hard-rejects a config still on `ai:` with a `charly migrate` hint. The independent kind:score `validate_ai_artifacts` flag is a separate concept, NOT renamed. Node-API, comment-preserving, idempotent. TouchesHost false (the per-host agent overlay is processed when `ctx.HostDeployPath` is set) |
| 2026.164.0001 | op-unify | the 2026-06 operation-unification cutover: collapse `task:` + the check list + the agent verb into one generic `Op` (verb × `do:` × `context:`); fold each candy's check list into `scenario:` as single-step scenarios; rewrite legacy `scope:`→`context:`. Comment-preserving, idempotent. (Its `do:` axis + `scenario:` are themselves superseded by the plan-unify step below.) |
| 2026.164.0003 | eval-check | rename the evaluation-harness verb `eval`→`check`: root `eval:`→`check:` registry, `eval_level`→`check_level`, `keep_eval_runs`→`keep_check_runs`, `kind: eval`→`kind: check` (schema-token renames only — author entity names are untouched). Node-API, comment-preserving, idempotent. |
| 2026.164.0005 | plan-unify | the plan-unification cutover: collapse the entire test/eval/benchmark surface into ONE flat `plan:` list. `task:` + `scenario:` → `plan:` (flattened); the `do:` axis + Gherkin `given/when/then/and/but` → the step keywords `run:`/`check:`/`agent-run:`/`agent-check:`; `setup:`/`teardown:`/`on_fail:`/`example:` dropped; `kind: recipe` + `from:` → `include:` steps; `kind: score` → an `iterate:` block on a `kind: check` bed; `description:` struct → string; the download `include:` filter → `extract_include:`. The loader's `rejectLegacyTestVocab` hard-rejects the old keys. Node-API, comment-preserving, idempotent. TouchesHost false. |
| 2026.165.1047 | sidecar-root | the 2026-06 embedded-config unification: `build.yml` + `sidecar.yml` consolidate into ONE embedded `charly/charly.yml` parsed by the same unified loader, and `sidecar:` becomes a first-class root key (`UnifiedFile.Sidecar`). Purely ADDITIVE (no-op, like `peer-field`): transforms nothing, raises HEAD so an older `charly` REJECTS a root-`sidecar:` config (with a `Run: charly migrate` hint) instead of silently dropping it. TouchesHost false. |
| **2026.165.1048** | **calver-schema** (HEAD) | **the universal stamper**: rewrite the top-level `version:` line of every versioned file to the HEAD CalVer (line-oriented, comment-preserving). This is the integer→CalVer transition (`version: 4` → the HEAD CalVer). **Always stays last** so `LatestSchemaVersion()` tracks it. |

Step `Name`s are for `--dry-run` / progress output only — they are no longer CLI sub-verbs. Steps that mutate per-host state (`~/.config/charly`, quadlets, `.secrets`) are flagged `TouchesHost`; see "Remote-cache auto-migration" below.

## How it runs

`RunMigrations(ctx)` walks `migrationSteps()` in order, calling each `Apply(ctx)`. Each step reports whether it changed anything; nothing-changed across the whole chain prints `nothing to migrate (already at schema <HEAD>)`. Per-step backups follow the established `<file>.bak.<unix-ts>` convention (field-singular, local-deploy, require-image, drop-kdbx, and the calver-schema stamp write them). `MigrateContext` (built by `NewMigrateContext`) carries the project dir, the per-host paths, the quadlet dir, the `.secrets` path, and `DryRun`.

### Remote-cache auto-migration (project-only)

`charly/refs.go` auto-runs `RunProjectMigrations(ctx)` on a freshly-cloned remote-repo cache so external repos pull through at the latest schema. `RunProjectMigrations` skips every `TouchesHost` step and leaves `HostDeployPath` empty, so a remote fetch **never mutates the user's per-host state** — even the calver-schema stamp touches only the cache's project files.

## Load-time gate

`LoadUnified` (`charly/unified.go`) parses the merged `version:` and rejects anything older than HEAD:

```
charly.yml: schema 2026.165.1048 is required (found "4"). Run: charly migrate
```

A non-CalVer value (the legacy integer `4`, empty, or garbage) parses as "older than every real CalVer", so a pre-CalVer config flows into the chain with no special case. Residual-key checks (e.g. `kind: deployment`, `target: host`, `secret_backend: kdbx`) remain as defense-in-depth, but **every** remediation hint now points uniformly at bare `charly migrate`.

## Adding a future cutover

1. Write the transform as an idempotent `Migrate*` function (its own file, like the existing migrators).
2. Append ONE `MigrationStep` to `migrationSteps()` with a CalVer **strictly greater** than the current HEAD. Keep the `calver-schema` stamp **last**.
3. Bump `latestSchemaVersion` to the new step's CalVer (the two are asserted equal by `TestRegistryHeadMatchesLatest`).
4. Update the HEAD-CalVer fixtures (the LoadUnified test fixtures + `testdata/*.yml` carry the stamp) and the repo's own versioned YAML.

The operator command never changes — it stays `charly migrate`.

### Standing rule: a schema/format change bumps `version:` AND mints a git tag

Any change to the YAML schema or composition format (a key rename, a deleted key, a new key shape — e.g. the 2026-05 `include:` → `import:` cutover) is a hard-cutover that MUST:

1. **Bump the `version:` CalVer** by appending a `MigrationStep` that raises `LatestSchemaVersion()` (the `calver-schema` stamp stays last). The load-time gate then rejects any not-yet-migrated config with a `Run: charly migrate` hint, so every reader sees the new format. Bumping `version:` WITHOUT a step that raises HEAD is forbidden — and `version:` is NEVER set above `LatestSchemaVersion()` (newer configs hard-fail at load).
2. **Mint a fresh per-push git tag** on the landing push — `v<YYYY.DDD.HHMM>` from the push moment (see "Per-push release git tags" above). The tag and the `version:` bump are decoupled: the tag marks the push, `version:` marks the schema. A schema cutover happens to do BOTH at once, but a content-only push (no schema change) still mints a tag at an unchanged `version:`.

## Idempotency

Running `charly migrate` twice is a no-op (`TestMigrateCalverSchema_StampsAndIdempotent`, plus each migrator's own idempotency tests). The registry invariants (HEAD == `LatestSchemaVersion()`, strictly-ascending order, unique names) are enforced by `charly/migrate_registry_test.go`.

## See Also

- `/charly-image:layer` — the `layer:` kind-keyed schema the chain produces
- `/charly-image:image` — `image:` entries + `charly box build/validate/inspect`
- `/charly-core:deploy` — `deploy:` entries the chain migrates (require-image, charly-cachyos)
- `/charly-local:local-spec` — `kind: local` templates (target-local, local-images)
- `/charly-build:secrets`, `/charly-build:settings` — credential schema (drop-kdbx, tailscale-secrets)
- `/charly-internals:go` — loader internals (`LoadUnified`, `ParseCalVer`, `RunMigrations`, `migrationSteps`)
- `/charly-internals:cutover-policy` — why hard-cutover + a single idempotent `charly migrate` is the required shape
