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
| **ov-core** | 15 | — | Lifecycle: start, stop, restart, ov-status, logs, shell, ssh, deploy, ov-update, remove, ov-config, cmd, ov-version, ov-doctor, clean. |
| **ov-build** | 13 | — | Build/authoring: build, generate, list, inspect, merge, new, pull, validate, secrets, settings, migrate, reconcile, ov-mcp-cmd. |
| **ov-eval** | 13 | — | Live-container evaluation: `eval` orchestrator + cdp, wl, wl-overlay, dbus, vnc, spice, libvirt, record, adb, appium probes + `android` (the `kind: android` device + `apk:` package format + `target: android` deploy) + the `eval-sway-browser-vnc-pod` R10 bed. |
| **ov-automation** | 6 | — | tmux verb, host-side wrappers (alias, udev), topic flags (enc, sidecar, openclaw-deploy). |

### kind — schema-kind authoring

| Plugin | Skill count | MCP server | Purpose |
|---|--:|---|---|
| **ov-image** | 2 | — | Schema for `kind: box` and `kind: candy` (box.yml / candy.yml authoring). |
| **ov-vm** | 8 | — | Schema for `kind: vm` + bootc VM catalog (cloud_image vs bootc, libvirt/QEMU). Includes `cachyos` (bootstrap VM, in the `overthinkos/cachyos` submodule) and `debian` / `ubuntu` (debootstrap bootstrap VMs, in the `overthinkos/debian` / `overthinkos/ubuntu` submodules). |
| **ov-kubernetes** | 2 | — | Schema for `kind: k8s` + cluster probes via `ov eval k8s`. |
| **ov-local** | 3 | — | Schema for `kind: local` + ssh-host deploys + managed ssh-config fragment. Includes `ov-cachyos` (operator CachyOS workstation profile, in the `overthinkos/cachyos` submodule). |
| **ov-pod** | 1 | — | Schema for `kind: pod` and `kind: deploy` — thin pointer to `/ov-core:deploy` for verb details. |

### development — contributor-only internals

| Plugin | Skill count | MCP server | Purpose |
|---|--:|---|---|
| **ov-internals** | 16 + 5 agents | github (stdio) | Go source map, install-plan IR, capabilities/OCI labels, vm-spec, libvirt/cloud-init renderers, cutover-policy, strict-policy, disposable, ovmf, generate-source, git-workflow, skills, agents (the agents/workflows/teams guide). Ships 5 agents — enforcers root-cause-analyzer, layer-validator, testing-validator; executors eval-bed-runner, deploy-verifier (drive the `ov eval` beds). The `/verify-beds` + `/audit-deploy-configs` dynamic workflows live in the superproject's `.claude/workflows/`. |

### images — deployable image catalog

#### Foundation layers (`ov-distros` / `ov-languages` / `ov-infrastructure` / `ov-tools`)

| Plugin | Skill count | MCP server | Purpose |
|---|--:|---|---|
| **ov-distros** | 41 | — | Base OS images, GPU runtime, bootc bootstrap, per-distro builders. fedora (+ fedora-builder, fedora-nonfree — the base stack stays in main's `base.yml`, imported by the `overthinkos/fedora` submodule under the `ov` namespace; the fedora-ov / fedora-test showcase images are owned by that submodule), arch (+ arch-builder, owned by the `overthinkos/arch` submodule), cachyos (+ cachyos-pacstrap/-builder, owned by the `overthinkos/cachyos` submodule), debian / ubuntu (+ their `-builder` and `-debootstrap`/`-debootstrap-builder`, owned by the `overthinkos/debian` / `overthinkos/ubuntu` submodules), aurora, bazzite, nvidia, cuda, rocm, bootc-base, bootc-config, etc. |
| **ov-languages** | 4 | — | Programming language runtimes — pixi, python, python-ml, python-ml-layer. (golang/rust/nodejs live in `ov-coder` because they're tightly coupled to dev images.) |
| **ov-infrastructure** | 22 | — | Databases, networking, security, system services. postgresql, redis, valkey, vectorchord, k3s, traefik, supervisord, tailscale, gocryptfs, virtualization, dbus-layer, tmux-layer, ssh-client, gnupg, etc. |
| **ov-tools** | 19 | — | CLI utilities and the `ov` binary — ripgrep, himalaya, whisper, nano-pdf, summarize, ordercli, gogcli, sherpa-onnx, songsee, blogwatcher, sag, xurl, goplaces, mcporter, yay, ujust, vscode, ov, ov-full. |

#### Per-pod plugins

| Plugin | Skill count | MCP server | Purpose |
|---|--:|---|---|
| **ov-jupyter** | 15 | jupyter @ 8888 | Jupyter image family (jupyter, jupyter-ml, jupyter-ml-notebook, unsloth-studio) + notebook templates + jupyter-mcp server. |
| **ov-coder** | 33 | ov @ 18765 | ov coder/dev images (fedora-coder in the `overthinkos/fedora` submodule; arch-coder/arch-ov in `overthinkos/arch`; debian-coder in `overthinkos/debian`; ubuntu-coder in `overthinkos/ubuntu`) + language runtimes (golang/rust/nodejs/docker-ce). |
| **ov-selkies** | 45 | chrome-devtools @ 9224 | Selkies-desktop family — labwc and full-KDE-Plasma flavors of the browser-streamed Wayland desktop, always a headless pod, per-GPU encode (VAAPI / NVENC / x264 auto-selected at runtime). |
| **ov-openclaw** | 7 | — | OpenClaw AI gateway family (CachyOS base): the `openclaw` layer + headless `openclaw` / `openclaw-full` images + the all-in-one `openclaw-desktop` (streaming desktop + gateway + CPU ollama + nested ov toolchain) + composition layers (`openclaw-full`, `openclaw-full-ml`). |
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
- `/ov-image:layer` — schema authoring for `kind: candy`.
- `/ov-distros:arch` — Arch Linux base image reference.
- `/ov-jupyter:notebook-templates` — bundled notebook starter content.
- `/ov-eval:cdp` — Chrome DevTools Protocol live probe.

## Skill name uniqueness

Every skill in this marketplace has a globally-unique folder name. Where a
short name could be ambiguous, the canonical names are:

- `ov-infrastructure:tmux-layer` (the tmux package layer) vs `ov-automation:tmux` (the verb).
- `ov-infrastructure:dbus-layer` (the D-Bus service layer) vs `ov-eval:dbus` (the verb).
- `ov-automation:openclaw-deploy` (the deployment topic) vs `ov-openclaw:openclaw` (the image).
- `ov-vm:vms-catalog` (the VM catalog skill) vs the kind-name `vm`.
- `ov-build:generate` (the build verb) vs `ov-internals:generate-source` (the source-reading reference).

## Recent changes

See the repo-root [`CHANGELOG.md`](../CHANGELOG.md) for the full history of
plugin reorganizations, skill renames, and marketplace version bumps. This
index and the skill docs themselves describe only the current structure.

## Installation

See [Claude Code plugin docs](https://code.claude.com/docs/en/discover-plugins)
for marketplace setup. To install plugins from this marketplace:

```bash
/plugin marketplace add overthinkos/overthink-plugins
/plugin install ov-core ov-jupyter         # for example: install just what you need
```

The `category:` field in `marketplace.json` lets the `/plugin` UI group
the listing by use-case bucket.
