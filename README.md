# Overthink Plugins

Claude Code plugins for the Overthink container image composition system.

## Which Plugins to Install

| Use Case | Plugins |
|----------|---------|
| **Building and running images** | overthink |
| **Contributing to the ov CLI** | overthink + overthink-dev |
| **Layer and image reference** | overthink-layers + overthink-images |

## Plugins

### overthink (16 skills)

Skills for composing, building, and running container images with the `ov` CLI.

| Skill | Invocation | Description |
|-------|-----------|-------------|
| layer | `/overthink:layer` | Layer authoring (layer.yml, root.yml, pixi.toml, etc.) |
| image | `/overthink:image` | Image composition (images.yml, defaults, inheritance) |
| build | `/overthink:build` | Building images (ov build, task build:*, caches) |
| run | `/overthink:run` | Runtime operations (ov shell, start/stop, aliases) |
| deploy | `/overthink:deploy` | Deployment (quadlet, bootc, tunnels, encryption) |
| validate | `/overthink:validate` | Validation rules and error handling (ov validate) |
| shell | `/overthink:shell` | Shell access (ov shell, --tty, -c, exec) |
| alias | `/overthink:alias` | Command aliases (ov alias add/remove/install) |
| config | `/overthink:config` | Runtime configuration (ov config get/set/list) |
| enc | `/overthink:enc` | Encrypted volumes (ov enc init/mount/unmount) |
| vm | `/overthink:vm` | Virtual machines (ov vm build/create/start/stop) |
| service | `/overthink:service` | Supervisord services (ov service start/stop/restart) |
| cdp | `/overthink:cdp` | Chrome DevTools Protocol (ov cdp open/list/click/eval) |
| vnc | `/overthink:vnc` | VNC desktop automation (ov vnc screenshot/click/type) |
| sway | `/overthink:sway` | Sway compositor control (ov sway focus/move/tree/exec) |
| openclaw | `/overthink:openclaw` | OpenClaw AI gateway configuration |

### overthink-dev (2 skills, 3 agents, GitHub MCP)

Development tools and enforcement agents for contributors.

**Skills:**

| Skill | Invocation | Description |
|-------|-----------|-------------|
| go | `/overthink-dev:go` | Go CLI development (build, test, code map) |
| generate | `/overthink-dev:generate` | Containerfile generation internals and debugging |

**Agents:**

| Agent | Type | Trigger |
|-------|------|---------|
| layer-validator | Blocking | Before editing layer.yml files |
| root-cause-analyzer | Blocking | Any error in output |
| testing-validator | Blocking | Claiming something "works" |

**MCP Server:** GitHub (22 tools for issues, PRs, workflows, repo operations)

### overthink-layers (61 skills)

Reference documentation for every Overthink container layer.

| Skill | Invocation | Description |
|-------|-----------|-------------|
| bootc-base | `/overthink-layers:bootc-base` | Bootc base composition (sshd + qemu-guest-agent + bootc-config) |
| bootc-config | `/overthink-layers:bootc-config` | Bootc system config (autologin, graphical target, pipewire) |
| build-toolchain | `/overthink-layers:build-toolchain` | C/C++ build tools (gcc, cmake, ninja, git) |
| chrome | `/overthink-layers:chrome` | Google Chrome with DevTools on :9222 |
| chrome-sway | `/overthink-layers:chrome-sway` | Chrome on Sway compositor via exec autostart |
| claude-code | `/overthink-layers:claude-code` | Claude Code CLI tool |
| cloud-init | `/overthink-layers:cloud-init` | Cloud instance initialization |
| comfyui | `/overthink-layers:comfyui` | ComfyUI image generation on :8188 |
| copr-desktop | `/overthink-layers:copr-desktop` | COPR desktop packages |
| cuda | `/overthink-layers:cuda` | CUDA toolkit + cuDNN + onnxruntime |
| dbus | `/overthink-layers:dbus` | D-Bus session bus |
| desktop-apps | `/overthink-layers:desktop-apps` | Chromium, VLC, KeePassXC, btop, cockpit, zsh |
| dev-tools | `/overthink-layers:dev-tools` | bat, ripgrep, neovim, gh, direnv, fd-find, htop |
| devops-tools | `/overthink-layers:devops-tools` | bind-utils, jq, rsync |
| docker-ce | `/overthink-layers:docker-ce` | Docker CE + buildx + compose |
| github-actions | `/overthink-layers:github-actions` | Act CLI via COPR + guestfs |
| github-runner | `/overthink-layers:github-runner` | GitHub Actions self-hosted runner |
| gocryptfs | `/overthink-layers:gocryptfs` | Encrypted filesystem for ov enc |
| golang | `/overthink-layers:golang` | Go compiler |
| google-cloud | `/overthink-layers:google-cloud` | Google Cloud SDK |
| google-cloud-npm | `/overthink-layers:google-cloud-npm` | GCP npm packages |
| grafana-tools | `/overthink-layers:grafana-tools` | Grafana tooling |
| immich | `/overthink-layers:immich` | Immich photo management on :2283 |
| immich-ml | `/overthink-layers:immich-ml` | Immich ML backend on :3003 |
| jupyter | `/overthink-layers:jupyter` | Jupyter + ML libs on :8888 |
| kubernetes | `/overthink-layers:kubernetes` | kubectl + Helm |
| language-runtimes | `/overthink-layers:language-runtimes` | Go, PHP, .NET, nodejs-devel, python3-devel |
| nodejs | `/overthink-layers:nodejs` | Node.js + npm via rpm/deb |
| nodejs24 | `/overthink-layers:nodejs24` | Node.js 24 via rpm/deb |
| ollama | `/overthink-layers:ollama` | Ollama LLM server on :11434 |
| openclaw | `/overthink-layers:openclaw` | OpenClaw AI gateway on :18789 |
| os-config | `/overthink-layers:os-config` | OS configuration |
| os-system-files | `/overthink-layers:os-system-files` | System files and configs |
| ov | `/overthink-layers:ov` | ov binary for container/VM use |
| ov-full | `/overthink-layers:ov-full` | ov + virtualization + gocryptfs + socat |
| pipewire | `/overthink-layers:pipewire` | Audio/media server + wireplumber |
| pixi | `/overthink-layers:pixi` | Pixi package manager + env/PATH |
| postgresql | `/overthink-layers:postgresql` | PostgreSQL server on :5432 |
| pre-commit | `/overthink-layers:pre-commit` | Git hooks framework |
| python | `/overthink-layers:python` | Python 3.13 via pixi |
| python-ml | `/overthink-layers:python-ml` | ML Python environment (PyTorch, transformers) |
| qemu-guest-agent | `/overthink-layers:qemu-guest-agent` | QEMU guest agent |
| redis | `/overthink-layers:redis` | Redis on :6379 |
| rpmfusion | `/overthink-layers:rpmfusion` | RPM Fusion repository configuration |
| rust | `/overthink-layers:rust` | Rust + Cargo via rpm/deb |
| socat | `/overthink-layers:socat` | Socket relay for port_relay and VM console |
| sshd | `/overthink-layers:sshd` | SSH server on :22 |
| supervisord | `/overthink-layers:supervisord` | Process manager via pixi |
| sway | `/overthink-layers:sway` | Sway Wayland compositor |
| sway-desktop | `/overthink-layers:sway-desktop` | Full desktop (pipewire + wayvnc + chrome-sway + terminal + thunar + waybar) |
| testapi | `/overthink-layers:testapi` | FastAPI test service on :9090 |
| thunar | `/overthink-layers:thunar` | Xfce file manager |
| traefik | `/overthink-layers:traefik` | Reverse proxy on :8000/:8080 |
| typst | `/overthink-layers:typst` | Document processor |
| ujust | `/overthink-layers:ujust` | Task runner |
| virtualization | `/overthink-layers:virtualization` | QEMU/KVM/libvirt stack |
| vr-streaming | `/overthink-layers:vr-streaming` | OpenXR, OpenVR, GStreamer |
| vscode | `/overthink-layers:vscode` | VS Code via Microsoft repo |
| waybar | `/overthink-layers:waybar` | Status bar + autotiling |
| wayvnc | `/overthink-layers:wayvnc` | VNC server on :5900 |
| xfce4-terminal | `/overthink-layers:xfce4-terminal` | Xfce terminal emulator |

### overthink-images (16 skills)

Reference documentation for every enabled Overthink container image.

| Skill | Invocation | Description |
|-------|-----------|-------------|
| fedora | `/overthink-images:fedora` | Base Fedora 43 image |
| fedora-nonfree | `/overthink-images:fedora-nonfree` | Fedora + RPM Fusion repos |
| fedora-builder | `/overthink-images:fedora-builder` | Builder with pixi, nodejs, build-toolchain |
| nvidia | `/overthink-images:nvidia` | CUDA GPU base image |
| python-ml | `/overthink-images:python-ml` | GPU ML Python environment |
| jupyter | `/overthink-images:jupyter` | Jupyter notebook + CUDA ML |
| ollama | `/overthink-images:ollama` | Ollama LLM server + CUDA |
| comfyui | `/overthink-images:comfyui` | ComfyUI image generation + CUDA |
| openclaw | `/overthink-images:openclaw` | Headless OpenClaw gateway |
| openclaw-ollama | `/overthink-images:openclaw-ollama` | OpenClaw + Ollama (headless) |
| openclaw-sway-browser | `/overthink-images:openclaw-sway-browser` | OpenClaw + desktop + Chrome |
| openclaw-ollama-sway-browser | `/overthink-images:openclaw-ollama-sway-browser` | OpenClaw + Ollama + desktop |
| githubrunner | `/overthink-images:githubrunner` | GitHub Actions self-hosted runner |
| immich | `/overthink-images:immich` | Immich photo management (CPU) |
| immich-cuda | `/overthink-images:immich-cuda` | Immich + CUDA ML backend |
| fedora-remote | `/overthink-images:fedora-remote` | Fedora with remote layer refs |

## Installation

### Via Team Configuration (settings.json)

```json
{
  "enabledPlugins": {
    "overthink@overthink-plugins": true,
    "overthink-dev@overthink-plugins": true,
    "overthink-layers@overthink-plugins": true,
    "overthink-images@overthink-plugins": true
  },
  "extraKnownMarketplaces": {
    "overthink-plugins": {
      "source": {
        "source": "directory",
        "path": "./plugins"
      }
    }
  }
}
```

## Component Totals

| Component | Count |
|-----------|-------|
| Plugins | 4 |
| Skills | 95 |
| Agents | 3 |
| MCP Servers | 1 (GitHub) |
| MCP Tools | 22 |
