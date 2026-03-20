# Overthink Plugins

Claude Code plugins for the Overthink container image composition system.

## Which Plugins to Install

| Use Case | Plugins |
|----------|---------|
| **Building and running images** | ov |
| **Contributing to the ov CLI** | ov + ov-dev |
| **Layer and image reference** | ov-layers + ov-images |

## Plugins

### ov (15 skills)

Skills for composing, building, and running container images with the `ov` CLI.

| Skill | Invocation | Description |
|-------|-----------|-------------|
| layer | `/ov:layer` | Layer authoring (layer.yml, root.yml, pixi.toml, etc.) |
| image | `/ov:image` | Image composition (images.yml, defaults, inheritance) |
| build | `/ov:build` | Building images (ov build, task build:*, caches) |
| deploy | `/ov:deploy` | Deployment (quadlet, bootc, tunnels, encryption) |
| validate | `/ov:validate` | Validation rules and error handling (ov validate) |
| shell | `/ov:shell` | Shell access (ov shell, --tty, -c, exec) |
| alias | `/ov:alias` | Command aliases (ov alias add/remove/install) |
| config | `/ov:config` | Runtime configuration (ov config get/set/list) |
| enc | `/ov:enc` | Encrypted volumes (ov enc init/mount/unmount) |
| vm | `/ov:vm` | Virtual machines (ov vm build/create/start/stop) |
| service | `/ov:service` | Service management (ov start/stop/enable + supervisord services) |
| cdp | `/ov:cdp` | Chrome DevTools Protocol (ov cdp open/list/click/eval) |
| vnc | `/ov:vnc` | VNC desktop automation (ov vnc screenshot/click/type) |
| sway | `/ov:sway` | Sway compositor control (ov sway focus/move/tree/exec) |
| openclaw | `/ov:openclaw` | OpenClaw AI gateway configuration |

### ov-dev (2 skills, 3 agents, GitHub MCP)

Development tools and enforcement agents for contributors.

**Skills:**

| Skill | Invocation | Description |
|-------|-----------|-------------|
| go | `/ov-dev:go` | Go CLI development (build, test, code map) |
| generate | `/ov-dev:generate` | Containerfile generation internals and debugging |

**Agents:**

| Agent | Type | Trigger |
|-------|------|---------|
| layer-validator | Blocking | Before editing layer.yml files |
| root-cause-analyzer | Blocking | Any error in output |
| testing-validator | Blocking | Claiming something "works" |

**MCP Server:** GitHub (22 tools for issues, PRs, workflows, repo operations)

### ov-layers (90 skills)

Reference documentation for every Overthink container layer.

| Skill | Invocation | Description |
|-------|-----------|-------------|
| blogwatcher | `/ov-layers:blogwatcher` | RSS monitor (depends: golang) |
| bootc-base | `/ov-layers:bootc-base` | Bootc base composition (sshd + qemu-guest-agent + bootc-config) |
| bootc-config | `/ov-layers:bootc-config` | Bootc system config (autologin, graphical target, pipewire) |
| build-toolchain | `/ov-layers:build-toolchain` | C/C++ build tools (gcc, cmake, ninja, git) |
| camsnap | `/ov-layers:camsnap` | Camera CLI (depends: golang) |
| chrome | `/ov-layers:chrome` | Google Chrome with DevTools on :9222 |
| chrome-sway | `/ov-layers:chrome-sway` | Chrome on Sway compositor via exec autostart |
| claude-code | `/ov-layers:claude-code` | Claude Code CLI tool |
| clawhub | `/ov-layers:clawhub` | ClawHub skill registry (depends: nodejs) |
| cloud-init | `/ov-layers:cloud-init` | Cloud instance initialization |
| codex | `/ov-layers:codex` | OpenAI Codex CLI (depends: nodejs) |
| comfyui | `/ov-layers:comfyui` | ComfyUI image generation on :8188 |
| copr-desktop | `/ov-layers:copr-desktop` | COPR desktop packages |
| cuda | `/ov-layers:cuda` | CUDA toolkit + cuDNN + onnxruntime |
| dbus | `/ov-layers:dbus` | D-Bus session bus |
| desktop-apps | `/ov-layers:desktop-apps` | Chromium, VLC, KeePassXC, btop, cockpit, zsh |
| dev-tools | `/ov-layers:dev-tools` | bat, ripgrep, neovim, gh, direnv, fd-find, htop |
| devops-tools | `/ov-layers:devops-tools` | bind-utils, jq, rsync |
| docker-ce | `/ov-layers:docker-ce` | Docker CE + buildx + compose |
| ffmpeg | `/ov-layers:ffmpeg` | FFmpeg multimedia framework |
| gemini | `/ov-layers:gemini` | Google Gemini CLI (depends: nodejs) |
| gh | `/ov-layers:gh` | GitHub CLI + git |
| gifgrep | `/ov-layers:gifgrep` | GIF search (depends: golang) |
| github-actions | `/ov-layers:github-actions` | Act CLI via COPR + guestfs |
| github-runner | `/ov-layers:github-runner` | GitHub Actions self-hosted runner |
| gocryptfs | `/ov-layers:gocryptfs` | Encrypted filesystem for ov enc |
| gogcli | `/ov-layers:gogcli` | Google Workspace CLI (depends: golang) |
| golang | `/ov-layers:golang` | Go compiler |
| google-cloud | `/ov-layers:google-cloud` | Google Cloud SDK |
| google-cloud-npm | `/ov-layers:google-cloud-npm` | GCP npm packages |
| goplaces | `/ov-layers:goplaces` | Google Places CLI (depends: golang) |
| grafana-tools | `/ov-layers:grafana-tools` | Grafana tooling |
| himalaya | `/ov-layers:himalaya` | Email CLI (depends: rust) |
| immich | `/ov-layers:immich` | Immich photo management on :2283 |
| immich-ml | `/ov-layers:immich-ml` | Immich ML backend on :3003 |
| jupyter | `/ov-layers:jupyter` | Jupyter + ML libs on :8888 |
| kubernetes | `/ov-layers:kubernetes` | kubectl + Helm |
| language-runtimes | `/ov-layers:language-runtimes` | Go, PHP, .NET, nodejs-devel, python3-devel |
| mcporter | `/ov-layers:mcporter` | MCP server CLI (depends: nodejs) |
| nano-pdf | `/ov-layers:nano-pdf` | PDF editor (depends: python) |
| nodejs | `/ov-layers:nodejs` | Node.js + npm via rpm/deb |
| nodejs24 | `/ov-layers:nodejs24` | Node.js 24 via rpm/deb |
| ollama | `/ov-layers:ollama` | Ollama LLM server on :11434 |
| openclaw | `/ov-layers:openclaw` | OpenClaw AI gateway on :18789 |
| openclaw-full | `/ov-layers:openclaw-full` | OpenClaw + Chrome + all OpenClaw tool layers |
| openclaw-full-ml | `/ov-layers:openclaw-full-ml` | openclaw-full + whisper + sherpa-onnx |
| oracle | `/ov-layers:oracle` | Prompt bundling CLI (depends: nodejs) |
| ordercli | `/ov-layers:ordercli` | Food delivery CLI (depends: golang) |
| os-config | `/ov-layers:os-config` | OS configuration |
| os-system-files | `/ov-layers:os-system-files` | System files and configs |
| ov | `/ov-layers:ov` | ov binary for container/VM use |
| ov-full | `/ov-layers:ov-full` | ov + virtualization + gocryptfs + socat |
| pipewire | `/ov-layers:pipewire` | Audio/media server + wireplumber |
| pixi | `/ov-layers:pixi` | Pixi package manager + env/PATH |
| playwright | `/ov-layers:playwright` | Browser automation (depends: nodejs) |
| postgresql | `/ov-layers:postgresql` | PostgreSQL server on :5432 |
| pre-commit | `/ov-layers:pre-commit` | Git hooks framework |
| python | `/ov-layers:python` | Python 3.13 via pixi |
| python-ml | `/ov-layers:python-ml` | ML Python environment (PyTorch, transformers) |
| qemu-guest-agent | `/ov-layers:qemu-guest-agent` | QEMU guest agent |
| redis | `/ov-layers:redis` | Redis on :6379 |
| ripgrep | `/ov-layers:ripgrep` | ripgrep search tool |
| rpmfusion | `/ov-layers:rpmfusion` | RPM Fusion repository configuration |
| rust | `/ov-layers:rust` | Rust + Cargo via rpm/deb |
| sag | `/ov-layers:sag` | ElevenLabs TTS (depends: golang) |
| sherpa-onnx | `/ov-layers:sherpa-onnx` | Offline TTS |
| socat | `/ov-layers:socat` | Socket relay for port_relay and VM console |
| songsee | `/ov-layers:songsee` | Audio spectrogram (depends: golang) |
| sqlite | `/ov-layers:sqlite` | SQLite database |
| sshd | `/ov-layers:sshd` | SSH server on :22 |
| summarize | `/ov-layers:summarize` | URL/transcript extractor (depends: nodejs) |
| supervisord | `/ov-layers:supervisord` | Process manager via pixi |
| sway | `/ov-layers:sway` | Sway Wayland compositor |
| sway-desktop | `/ov-layers:sway-desktop` | Full desktop (pipewire + wayvnc + chrome-sway + terminal + thunar + waybar) |
| testapi | `/ov-layers:testapi` | FastAPI test service on :9090 |
| thunar | `/ov-layers:thunar` | Xfce file manager |
| tmux | `/ov-layers:tmux` | Terminal multiplexer |
| traefik | `/ov-layers:traefik` | Reverse proxy on :8000/:8080 |
| typst | `/ov-layers:typst` | Document processor |
| ujust | `/ov-layers:ujust` | Task runner |
| uv | `/ov-layers:uv` | Python package manager (depends: python) |
| virtualization | `/ov-layers:virtualization` | QEMU/KVM/libvirt stack |
| vr-streaming | `/ov-layers:vr-streaming` | OpenXR, OpenVR, GStreamer |
| vscode | `/ov-layers:vscode` | VS Code via Microsoft repo |
| wacli | `/ov-layers:wacli` | WhatsApp CLI (depends: golang) |
| waybar | `/ov-layers:waybar` | Status bar + autotiling |
| wayvnc | `/ov-layers:wayvnc` | VNC server on :5900 |
| whisper | `/ov-layers:whisper` | Local STT (depends: python, cuda) |
| xfce4-terminal | `/ov-layers:xfce4-terminal` | Xfce terminal emulator |
| xurl | `/ov-layers:xurl` | X/Twitter API CLI (depends: nodejs) |

### ov-images (16 skills)

Reference documentation for every enabled Overthink container image.

| Skill | Invocation | Description |
|-------|-----------|-------------|
| fedora | `/ov-images:fedora` | Base Fedora 43 image |
| fedora-nonfree | `/ov-images:fedora-nonfree` | Fedora + RPM Fusion repos |
| fedora-builder | `/ov-images:fedora-builder` | Builder with pixi, nodejs, build-toolchain |
| nvidia | `/ov-images:nvidia` | CUDA GPU base image |
| python-ml | `/ov-images:python-ml` | GPU ML Python environment |
| jupyter | `/ov-images:jupyter` | Jupyter notebook + CUDA ML |
| ollama | `/ov-images:ollama` | Ollama LLM server + CUDA |
| comfyui | `/ov-images:comfyui` | ComfyUI image generation + CUDA |
| openclaw | `/ov-images:openclaw` | Headless OpenClaw gateway |
| openclaw-ollama | `/ov-images:openclaw-ollama` | OpenClaw + Ollama (headless) |
| openclaw-sway-browser | `/ov-images:openclaw-sway-browser` | OpenClaw + desktop + Chrome |
| openclaw-ollama-sway-browser | `/ov-images:openclaw-ollama-sway-browser` | OpenClaw + Ollama + desktop |
| githubrunner | `/ov-images:githubrunner` | GitHub Actions self-hosted runner |
| immich | `/ov-images:immich` | Immich photo management (CPU) |
| immich-cuda | `/ov-images:immich-cuda` | Immich + CUDA ML backend |
| fedora-remote | `/ov-images:fedora-remote` | Fedora with remote layer refs |

## Installation

### Via Team Configuration (settings.json)

```json
{
  "enabledPlugins": {
    "ov@ov-plugins": true,
    "ov-dev@ov-plugins": true,
    "ov-layers@ov-plugins": true,
    "ov-images@ov-plugins": true
  },
  "extraKnownMarketplaces": {
    "ov-plugins": {
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
| Skills | 123 |
| Agents | 3 |
| MCP Servers | 1 (GitHub) |
| MCP Tools | 22 |
