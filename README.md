# Overthink Plugins

Claude Code plugins for Overthink — the container management experience for you and your AI.

## How this marketplace is organized

Plugins are sorted into **four use-case buckets**:

| Bucket | When to install | Plugins |
|---|---|---|
| **commands** | "I want to run ov verbs" | `ov-core`, `ov-build`, `ov-eval`, `ov-automation` |
| **kind** | "I want to author the YAML schema for an entity" | `ov-image`, `ov-vm`, `ov-kubernetes`, `ov-local`, `ov-pod` |
| **development** | "I'm a contributor working on the ov source code itself" | `ov-internals` |
| **images** | "I want to deploy a specific image" | `ov-distros`, `ov-languages`, `ov-infrastructure`, `ov-tools`, `ov-jupyter`, `ov-coder`, `ov-selkies`, `ov-openclaw`, `ov-versa`, `ov-ollama`, `ov-openwebui`, `ov-comfyui`, `ov-immich`, `ov-hermes`, `ov-filebrowser` |

The directory layout under `plugins/` is **flat** — every plugin sits at
`plugins/<name>/` (no `ov-` prefix in directory names). The `ov-` prefix
lives exclusively in each `plugin.json`'s `name:` field, which means every
skill invocation is `/ov-<plugin>:<skill>` (e.g. `/ov-core:ssh`,
`/ov-jupyter:jupyter`, `/ov-distros:arch`). The `category:` field in
`marketplace.json` provides the four-bucket grouping for the plugin
manager UI.

## Plugins by bucket

### commands — runtime CLI verbs

| Plugin | Skill count | MCP server | Purpose |
|---|--:|---|---|
| **ov-core** | 14 | — | Lifecycle: start, stop, restart, ov-status, logs, shell, ssh, deploy, ov-update, remove, ov-config, cmd, ov-version, ov-doctor. |
| **ov-build** | 12 | — | Build/authoring: build, generate, list, inspect, merge, new, pull, validate, secrets, settings, migrate, ov-mcp-cmd. |
| **ov-eval** | 12 | — | Live-container evaluation: `eval` orchestrator + cdp, wl, wl-overlay, dbus, vnc, spice, libvirt, record, adb, appium probes + the `eval-sway-browser-vnc` R10 bed. |
| **ov-automation** | 6 | — | tmux verb, host-side wrappers (alias, udev), topic flags (enc, sidecar, openclaw-deploy). |

### kind — schema-kind authoring

| Plugin | Skill count | MCP server | Purpose |
|---|--:|---|---|
| **ov-image** | 2 | — | Schema for `kind: image` and `kind: layer` (image.yml / layer.yml authoring). |
| **ov-vm** | 10 | — | Schema for `kind: vm` + bootc VM catalog (cloud_image vs bootc, libvirt/QEMU). Includes `cachyos` (bootstrap VM, in the `overthinkos/cachyos` submodule) and `debian` / `ubuntu` (debootstrap bootstrap VMs, in the `overthinkos/debian` / `overthinkos/ubuntu` submodules). |
| **ov-kubernetes** | 2 | — | Schema for `kind: k8s` + cluster probes via `ov eval k8s`. |
| **ov-local** | 3 | — | Schema for `kind: local` + ssh-host deploys + managed ssh-config fragment. Includes `ov-cachyos` (operator CachyOS workstation profile, in the `overthinkos/cachyos` submodule). |
| **ov-pod** | 1 | — | Schema for `kind: pod` and `kind: deploy` — thin pointer to `/ov-core:deploy` for verb details. |

### development — contributor-only internals

| Plugin | Skill count | MCP server | Purpose |
|---|--:|---|---|
| **ov-internals** | 14 + 3 agents | github (stdio) | Go source map, install-plan IR, capabilities/OCI labels, vm-spec, libvirt/cloud-init renderers, cutover-policy, strict-policy, disposable, ovmf, generate-source, skills. Ships enforcement agents (root-cause-analyzer, layer-validator, testing-validator). |

### images — deployable image catalog

#### Foundation layers (split from the old `ov-foundation` mega-plugin)

| Plugin | Skill count | MCP server | Purpose |
|---|--:|---|---|
| **ov-distros** | 40 | — | Base OS images, GPU runtime, bootc bootstrap, per-distro builders. fedora (+ fedora-builder, fedora-nonfree — the base stack stays in main's `fedora-base.yml`; the fedora-ov / fedora-test showcase images are owned by the `overthinkos/fedora` submodule), arch (+ arch-builder, owned by the `overthinkos/arch` submodule), cachyos (+ cachyos-pacstrap/-builder, owned by the `overthinkos/cachyos` submodule), debian / ubuntu (+ their `-builder` and `-debootstrap`/`-debootstrap-builder`, owned by the `overthinkos/debian` / `overthinkos/ubuntu` submodules), aurora, bazzite, nvidia, cuda, rocm, bootc-base, bootc-config, etc. |
| **ov-languages** | 4 | — | Programming language runtimes — pixi, python, python-ml, python-ml-layer. (golang/rust/nodejs live in `ov-coder` because they're tightly coupled to dev images.) |
| **ov-infrastructure** | 22 | — | Databases, networking, security, system services. postgresql, redis, valkey, vectorchord, k3s, traefik, supervisord, tailscale, gocryptfs, virtualization, dbus-layer, tmux-layer, ssh-client, gnupg, etc. |
| **ov-tools** | 19 | — | CLI utilities and the `ov` binary — ripgrep, himalaya, whisper, nano-pdf, summarize, ordercli, gogcli, sherpa-onnx, songsee, blogwatcher, sag, xurl, goplaces, mcporter, yay, ujust, vscode, ov, ov-full. |

#### Per-pod plugins

| Plugin | Skill count | MCP server | Purpose |
|---|--:|---|---|
| **ov-jupyter** | 15 | jupyter @ 8888 | Jupyter image family (jupyter, jupyter-ml, jupyter-ml-notebook, unsloth-studio) + notebook templates + jupyter-mcp server. |
| **ov-coder** | 33 | ov @ 18765 | ov coder/dev images (fedora-coder in the `overthinkos/fedora` submodule; arch-coder/arch-ov in `overthinkos/arch`; debian-coder in `overthinkos/debian`; ubuntu-coder in `overthinkos/ubuntu`) + language runtimes (golang/rust/nodejs/docker-ce). |
| **ov-selkies** | 41 | chrome-devtools @ 9224 | Selkies-desktop family (Wayland in container with VNC + Chrome). |
| **ov-openclaw** | 7 | chrome-devtools @ 9224 | OpenClaw AI gateway family: the `openclaw` layer + headless `openclaw`/`openclaw-ollama` images + composition layers (`openclaw-full`, `openclaw-full-ml`). |
| **ov-versa** | 9 | marimo @ 22718, airflow @ 29999 | Versa image — marimo notebook + Airflow + OSM/GTFS analytics + martin vector tiles. |
| **ov-ollama** | 2 | — | Ollama LLM-server image. Pair with `ov-jupyter` to expose to notebooks. |
| **ov-openwebui** | 2 | — | OpenWebUI chat frontend. Consumes the jupyter MCP. |
| **ov-comfyui** | 2 | — | ComfyUI image-generation/diffusion. |
| **ov-immich** | 4 | — | Immich photo-management (immich + immich-ml variants). |
| **ov-hermes** | 6 | — | Hermes agent image (hermes + hermes-playwright variants). Consumes the jupyter MCP. |
| **ov-filebrowser** | 2 | — | Filebrowser web file management on top of an ov volume. |

## Skill invocation pattern

Every skill uses the namespaced form `/<plugin-name>:<skill-name>` per the
[Claude Code plugin docs](https://code.claude.com/docs/en/plugins). The
plugin name carries the `ov-` prefix; the skill name does not. Examples:

- `/ov-core:ssh` — open an interactive shell into a pod.
- `/ov-image:layer` — schema authoring for `kind: layer`.
- `/ov-distros:arch` — Arch Linux base image reference.
- `/ov-jupyter:notebook-templates` — bundled notebook starter content.
- `/ov-eval:cdp` — Chrome DevTools Protocol live probe.

## Skill name uniqueness

Every skill in this marketplace has a globally-unique folder name. Five
former collisions were resolved during the 2026-05-XX reorganization:

- `tmux` (was a foundation layer + an advanced verb) → foundation's
  package layer is now `ov-infrastructure:tmux-layer`; the verb stays
  `ov-automation:tmux`.
- `dbus` (was a foundation layer + an advanced verb) → foundation's
  service layer is now `ov-infrastructure:dbus-layer`; the verb stays
  `ov-eval:dbus`.
- `openclaw` (was an advanced topic + an image plugin's image skill) →
  the topic is now `ov-automation:openclaw-deploy`; the image keeps
  `ov-openclaw:openclaw`.
- `vms` (was the catalog skill in the old `ov-vms` plugin) → renamed to
  `ov-vm:vms-catalog` for clarity vs the kind-name `vm`.
- `generate` (was a build verb + a dev source-reading reference) → the
  verb stays `ov-build:generate`; the dev reference is now
  `ov-internals:generate-source`.

## Recent changes

- **2026-05-XX** — Use-case reorganization (this cutover). Plugins
  sorted into four buckets (commands / kind / development / images).
  `ov-foundation` (79-skill mega-plugin) split into `ov-distros` /
  `ov-languages` / `ov-infrastructure` / `ov-tools`. `ov-vms` folded into
  `ov-vm`. `ov-advanced` retired; its skills split between `ov-eval`
  (probes), `ov-automation` (topic flags + tmux/alias/udev), and the
  kind plugins (`ov-vm`, `ov-kubernetes`, `ov-local`). `ov-build`
  schema-authoring skills (`image`, `layer`, `local-spec`) moved to
  dedicated kind plugins; `ov-build:eval` orchestrator moved to
  `ov-eval`. `ov-dev` renamed to `ov-internals`. Directory names
  dropped the `ov-` prefix while plugin.json `name:` fields kept it;
  result is the same `/ov-<plugin>:<skill>` invocation surface with a
  cleaner `ls plugins/`. Skill-name collisions (`tmux`, `dbus`,
  `openclaw`, `vms`, `generate`) renamed for global uniqueness.
- **2026-05-05** — Engineering-discipline cutover. R1–R10 reordered.
  R1–R5 = engineering discipline; R6–R9 = runtime verification; R10 =
  final acceptance. New skill `/ov-internals:strict-policy`. AI
  Attribution: any rule violation FORBIDS commit at any tier.
- **2026-05-XX** — Init-system polymorphism. The `*-host` sibling-layer
  pattern (`virtualization-host`, `ov-full-host`) deleted; unified
  layers carry both supervisord and systemd renders via mixed `service:`
  entries.
- **2026-05-05** — Cross-kind name reuse + `overthink.yml`-only
  authoring. Same name MAY exist across layer / image / pod / vm / k8s /
  local / deploy. All authoring verbs default to `overthink.yml`.
- **2026-04-30** — Three-tier marketplace v2.0.0: `ov-core` / `ov-build` /
  `ov-advanced` (split from the original monolithic `ov` plugin) +
  per-pod plugins + `ov-foundation` reference. (This 2026-05-XX cutover
  supersedes that structure.)

## Installation

See [Claude Code plugin docs](https://code.claude.com/docs/en/discover-plugins)
for marketplace setup. To install plugins from this marketplace:

```bash
/plugin marketplace add overthinkos/overthink-plugins
/plugin install ov-core ov-jupyter         # for example: install just what you need
```

The `category:` field in `marketplace.json` lets the `/plugin` UI group
the listing by use-case bucket.
