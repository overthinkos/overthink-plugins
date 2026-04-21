# Overthink Plugins

Claude Code plugins for Overthink — the container management experience for you and your AI.

## Which Plugins to Install

| Use Case | Plugins |
|----------|---------|
| **Building and running images** | ov |
| **Contributing to the ov CLI** | ov + ov-dev |
| **Layer and image reference** | ov-layers + ov-images |
| **Programmatic notebook access** | ov-jupyter |

## Plugins

### ov (39 skills)

Skills for composing, building, and running container images with the `ov` CLI.

Build-mode commands live under `ov image …` (the only family that reads
`image.yml`). Every other command reads exclusively from OCI labels +
`deploy.yml`. See `/ov:image` for the family overview.

| Skill | Invocation | Description |
|-------|-----------|-------------|
| alias | `/ov:alias` | Command aliases (ov alias add/remove/install) |
| build | `/ov:build` | `ov image build` — building images, caches |
| cdp | `/ov:cdp` | Chrome DevTools Protocol (ov test cdp open/list/click/eval) |
| cmd | `/ov:cmd` | Single command execution with D-Bus notification |
| config | `/ov:config` | Unified setup: quadlet + secrets + volumes + data provisioning |
| dbus | `/ov:dbus` | D-Bus interaction inside containers (notify, call, list, introspect) |
| deploy | `/ov:deploy` | Deployment verb family: `ov deploy add/del` (unified; host + container targets), deploy.yml management, quadlet, tunnels, volume backing |
| host-deploy | `/ov:host-deploy` | Host-target deploys (`ov deploy add host`): ledger, 15 ReverseOps, `--with-services`/`--allow-repo-changes`/`--allow-root-tasks` gates, systemd rendering |
| doctor | `/ov:doctor` | Host dependency and hardware checks |
| enc | `/ov:enc` | Encrypted volumes (ov config mount/unmount/status/passwd) |
| generate | `/ov:generate` | `ov image generate` — Containerfile generation from image.yml and layers |
| image | `/ov:image` | `ov image` family overview + image.yml composition reference |
| inspect | `/ov:inspect` | `ov image inspect` — resolved config as JSON |
| layer | `/ov:layer` | Layer authoring (layer.yml with tasks:, pixi.toml, etc.) |
| list | `/ov:list` | `ov image list` — images, layers, targets, services, routes, volumes, aliases |
| logs | `/ov:logs` | Service log viewing (ov logs, -f for follow) |
| merge | `/ov:merge` | `ov image merge` — post-build layer optimization |
| new | `/ov:new` | `ov image new layer` — scaffold new layers |
| openclaw | `/ov:openclaw` | OpenClaw AI gateway configuration |
| pull | `/ov:pull` | `ov image pull` — fetch into local storage; ErrImageNotLocal recovery |
| record | `/ov:record` | Recording sessions (ov record start/stop/list/cmd) |
| remove | `/ov:remove` | Remove service container, quadlet, and deploy.yml entry |
| secrets | `/ov:secrets` | KeePass .kdbx and GPG secret management |
| service | `/ov:service` | Init system service management inside containers |
| settings | `/ov:settings` | Runtime configuration (ov settings get/set/list/reset) |
| shell | `/ov:shell` | Shell access (ov shell, --tty, -c, exec) |
| sidecar | `/ov:sidecar` | Sidecar containers, pod networking, Tailscale exit nodes |
| start | `/ov:start` | Start container as background service |
| status | `/ov:status` | Service status with tool probes and device detection |
| stop | `/ov:stop` | Stop running container |
| tmux | `/ov:tmux` | Persistent tmux sessions (ov tmux shell/cmd/run/attach/capture) |
| udev | `/ov:udev` | GPU device access rules (ov udev status/generate/install/remove) |
| update | `/ov:update` | Update image and restart with data sync |
| validate | `/ov:validate` | `ov image validate` — image.yml + layer definitions |
| version | `/ov:version` | Show CLI version information |
| vm | `/ov:vm` | Virtual machines (ov vm build/create/start/stop) |
| vnc | `/ov:vnc` | VNC desktop automation (ov test vnc screenshot/click/type) |
| wl | `/ov:wl` | Desktop automation (22 commands + 12 sway IPC commands) |
| wl-overlay | `/ov:wl-overlay` | Wayland overlays for screen recordings |

### ov-dev (4 skills, 3 agents, GitHub MCP)

Development tools and enforcement agents for contributors.

**Skills:**

| Skill | Invocation | Description |
|-------|-----------|-------------|
| go | `/ov-dev:go` | Go CLI development (build, test, code map, mode purity, self-exec, IR refactor pointer) |
| generate | `/ov-dev:generate` | Containerfile generation internals and debugging; OCITarget + IR integration |
| install-plan | `/ov-dev:install-plan` | The shared InstallPlan IR: 8 step kinds, DeployTarget interface, BuildDeployPlan compiler, OCITarget/ContainerDeployTarget/HostDeployTarget |
| host-infra | `/ov-dev:host-infra` | Host-deploy supporting files: hostdistro, install_ledger, builder_run, shell_profile, reverse_ops, service_render, deploy_ref, migrate_services_tool |

**Agents:**

| Agent | Type | Trigger |
|-------|------|---------|
| layer-validator | Blocking | Before editing layer.yml files |
| root-cause-analyzer | Blocking | Any error in output |
| testing-validator | Blocking | Claiming something "works" |

**MCP Server:** GitHub (22 tools for issues, PRs, workflows, repo operations)

### ov-jupyter (MCP server)

Jupyter MCP server for programmatic notebook access with real-time collaboration.

**MCP Server:** Streamable HTTP at `http://localhost:8888/mcp` (when jupyter or jupyter-ml container is running).

| Tool | Description |
|------|-------------|
| list_notebooks | List all notebooks in the workspace |
| get_notebook | Read full notebook content |
| create_notebook | Create a new notebook |
| open_notebook_session | Open a CRDT collaboration session |
| close_notebook_session | Close a collaboration session |
| get_cell | Read a specific cell |
| update_cell | Modify cell source or type |
| insert_cell | Add a new cell at position |
| delete_cell | Remove a cell |
| execute_cell | Run a cell and return output |
| watch_notebook | Watch for real-time changes |
| get_active_sessions | List open collaboration sessions |
| get_active_users | List connected collaborators |

Multiple MCP clients can edit the same notebook simultaneously — changes sync via CRDT.

### ov-layers (160 skills)

Reference documentation for every Overthink container layer.

| Skill | Invocation | Description |
|-------|-----------|-------------|
| a11y-tools | `/ov-layers:a11y-tools` | AT-SPI2 accessibility introspection for element-based automation |
| agent-forwarding | `/ov-layers:agent-forwarding` | SSH/GPG agent forwarding (gnupg + direnv + ssh-client) |
| arch-aur-test | `/ov-layers:arch-aur-test` | AUR package installation test layer for Arch Linux |
| arch-pac-test | `/ov-layers:arch-pac-test` | Pacman package installation test layer for Arch Linux |
| asciinema | `/ov-layers:asciinema` | Terminal session recorder (asciinema) |
| blogwatcher | `/ov-layers:blogwatcher` | Blog/RSS feed monitor CLI |
| bootc-base | `/ov-layers:bootc-base` | Base composition for bootc OS images (sshd + qemu-guest-agent + bootc-config) |
| bootc-config | `/ov-layers:bootc-config` | Bootc system config (autologin, graphical target, pipewire) |
| build-toolchain | `/ov-layers:build-toolchain` | C/C++ build toolchain (gcc, cmake, autoconf, ninja, git, pkg-config) |
| camsnap | `/ov-layers:camsnap` | RTSP/ONVIF camera snapshot and clip CLI |
| chrome | `/ov-layers:chrome` | Google Chrome with DevTools on port 9222 |
| chrome-kwin | `/ov-layers:chrome-kwin` | Google Chrome on KWin compositor with DevTools protocol |
| chrome-mutter | `/ov-layers:chrome-mutter` | Google Chrome on Mutter compositor with DevTools protocol |
| chrome-niri | `/ov-layers:chrome-niri` | Google Chrome on Niri compositor with DevTools protocol |
| chrome-sway | `/ov-layers:chrome-sway` | Google Chrome on Sway compositor via exec autostart with DevTools protocol |
| chrome-x11 | `/ov-layers:chrome-x11` | Google Chrome on X11 with DevTools protocol via Openbox autostart |
| claude-code | `/ov-layers:claude-code` | Claude Code CLI installed globally via npm |
| clawhub | `/ov-layers:clawhub` | ClawHub CLI for searching and installing OpenClaw skills |
| cloud-init | `/ov-layers:cloud-init` | Cloud-init for instance initialization with NoCloud datasource |
| codex | `/ov-layers:codex` | OpenAI Codex CLI coding agent |
| container-nesting | `/ov-layers:container-nesting` | Container-in-container support (podman, buildah, fuse-overlayfs) |
| comfyui | `/ov-layers:comfyui` | ComfyUI image generation service on port 8188 with CUDA |
| copr-desktop | `/ov-layers:copr-desktop` | COPR desktop packages (CoolerControl, Ghostty, Nerd Fonts, WinBoat) |
| cuda | `/ov-layers:cuda` | CUDA toolkit, cuDNN, ONNX Runtime from negativo17 repos |
| dbus | `/ov-layers:dbus` | D-Bus session bus for IPC and `ov test dbus` commands |
| desktop-apps | `/ov-layers:desktop-apps` | Chromium, VLC, KeePassXC, btop, cockpit, zsh |
| desktop-fonts | `/ov-layers:desktop-fonts` | JetBrains Mono and Nerd Fonts for desktop containers |
| dev-tools | `/ov-layers:dev-tools` | bat, ripgrep, neovim, gh, direnv, fd-find, htop |
| devops-tools | `/ov-layers:devops-tools` | AWS CLI, Scaleway, kubectx/kubens, OpenTofu, wrangler, bind-utils, jq, rsync |
| direnv | `/ov-layers:direnv` | Automatic environment variable loading from .envrc files |
| docker-ce | `/ov-layers:docker-ce` | Docker CE engine with buildx and compose plugins |
| fastfetch | `/ov-layers:fastfetch` | Fast system information tool (neofetch successor) |
| ffmpeg | `/ov-layers:ffmpeg` | FFmpeg multimedia framework |
| gemini | `/ov-layers:gemini` | Google Gemini CLI for AI coding assistance and search |
| gh | `/ov-layers:gh` | GitHub CLI and git |
| gifgrep | `/ov-layers:gifgrep` | GIF search and download CLI |
| github-actions | `/ov-layers:github-actions` | Act CLI via COPR + guestfs-tools |
| github-runner | `/ov-layers:github-runner` | GitHub Actions self-hosted runner as supervised service |
| gnupg | `/ov-layers:gnupg` | GnuPG encryption and signing tools for GPG agent forwarding |
| gocryptfs | `/ov-layers:gocryptfs` | Encrypted filesystem for ov config encrypted volume operations |
| gogcli | `/ov-layers:gogcli` | Google Workspace CLI (Gmail, Calendar, Drive, Contacts, Sheets, Docs) |
| golang | `/ov-layers:golang` | Go programming language compiler |
| google-cloud | `/ov-layers:google-cloud` | Google Cloud SDK (gcloud, gsutil, bq) |
| google-cloud-npm | `/ov-layers:google-cloud-npm` | GCP npm packages (firebase-tools, Gemini CLI) |
| goplaces | `/ov-layers:goplaces` | Google Places API CLI for location search |
| grafana-tools | `/ov-layers:grafana-tools` | Grafana observability CLI tools (logcli, promtool, mimirtool, tanka) |
| heroic | `/ov-layers:heroic` | Heroic Games Launcher for Epic, GOG, and Amazon Prime Gaming |
| himalaya | `/ov-layers:himalaya` | Himalaya email CLI (IMAP/SMTP) |
| immich | `/ov-layers:immich` | Immich photo management server on port 2283 |
| immich-ml | `/ov-layers:immich-ml` | Immich ML backend on port 3003 |
| jupyter | `/ov-layers:jupyter` | Jupyter notebook server on port 8888 with CUDA and ML libraries |
| jupyter | `/ov-layers:jupyter` | JupyterLab with real-time collaboration + MCP server on port 8888 |
| jupyter-ml | `/ov-layers:jupyter-ml` | Full CUDA ML + JupyterLab collaboration + CRDT MCP on port 8888 |
| kubernetes | `/ov-layers:kubernetes` | kubectl and Helm package manager |
| kwin | `/ov-layers:kwin` | KWin Wayland compositor running headless with virtual backend |
| kwin-apps | `/ov-layers:kwin-apps` | KDE-native desktop applications (Konsole, Dolphin) |
| kwin-desktop | `/ov-layers:kwin-desktop` | KDE desktop composition (KWin, PipeWire, XDG Portal, Chrome, Konsole, Dolphin) |
| labwc | `/ov-layers:labwc` | Lightweight wlroots-based Wayland compositor for nested desktop in pixelflux |
| language-runtimes | `/ov-layers:language-runtimes` | Go, PHP, .NET, nodejs-devel, python3-devel |
| libnotify | `/ov-layers:libnotify` | Desktop notification client (notify-send CLI) |
| llama-cpp | `/ov-layers:llama-cpp` | llama.cpp prebuilt binaries and GGUF conversion tools |
| mcporter | `/ov-layers:mcporter` | MCP server CLI for listing, configuring, and calling MCP tools |
| mutter | `/ov-layers:mutter` | GNOME Mutter Wayland compositor running headless with virtual monitor |
| mutter-apps | `/ov-layers:mutter-apps` | GNOME-native desktop applications (gnome-terminal, Nautilus) |
| mutter-desktop | `/ov-layers:mutter-desktop` | GNOME desktop composition (Mutter, PipeWire, XDG Portal, Chrome, gnome-terminal, Nautilus) |
| nano-pdf | `/ov-layers:nano-pdf` | nano-pdf CLI for PDF editing with natural language |
| niri | `/ov-layers:niri` | Niri Wayland compositor (Smithay-based) built from source with virtual output |
| niri-apps | `/ov-layers:niri-apps` | Desktop applications (terminal, file manager) for Niri compositor |
| niri-desktop | `/ov-layers:niri-desktop` | Niri desktop composition with audio, portals, Chrome, terminal, and file manager |
| nvidia | `/ov-layers:nvidia` | NVIDIA GPU runtime support: driver libs, nvidia-container-toolkit (CDI), VA-API |
| nodejs | `/ov-layers:nodejs` | Node.js and npm via system packages (RPM/DEB) |
| nodejs24 | `/ov-layers:nodejs24` | Node.js 24 and npm via Fedora RPM packages |
| notebook-finetuning | `/ov-layers:notebook-finetuning` | Unsloth fine-tuning notebook collection (data layer) |
| notebook-llm-on-supercomputers | `/ov-layers:notebook-llm-on-supercomputers` | LLMs on Supercomputers course notebooks (data layer) |
| notebook-ollama | `/ov-layers:notebook-ollama` | Ollama integration notebook collection (data layer) |
| notebook-templates | `/ov-layers:notebook-templates` | Starter notebook templates (data layer for jupyter) |
| ollama | `/ov-layers:ollama` | Ollama LLM server on port 11434 with CUDA and model persistence |
| openbox | `/ov-layers:openbox` | Openbox lightweight X11 window manager with keybindings |
| openclaw | `/ov-layers:openclaw` | OpenClaw AI gateway service on port 18789 |
| openclaw-full | `/ov-layers:openclaw-full` | Maximal OpenClaw deployment (gateway + browser + all tools/skills) |
| openclaw-full-ml | `/ov-layers:openclaw-full-ml` | OpenClaw full + ML tools (whisper, sherpa-onnx, CUDA) |
| openwebui | `/ov-layers:openwebui` | Open WebUI with auto-configured LLM providers, MCP servers, and Jupyter |
| oracle | `/ov-layers:oracle` | Oracle CLI for prompt bundling and multi-engine AI queries |
| ordercli | `/ov-layers:ordercli` | Food delivery order status CLI (Foodora) |
| os-config | `/ov-layers:os-config` | OS system configuration for bootc images |
| os-system-files | `/ov-layers:os-system-files` | System files overlay and justfile imports for bootc images |
| ov | `/ov-layers:ov` | Overthink CLI (ov) binary for container/VM use |
| ov-full | `/ov-layers:ov-full` | Full ov toolchain (CLI + virtualization + gocryptfs + socat) |
| pavucontrol | `/ov-layers:pavucontrol` | PulseAudio volume control GUI for desktop containers |
| pipewire | `/ov-layers:pipewire` | PipeWire audio/media server with WirePlumber session manager |
| pixi | `/ov-layers:pixi` | Pixi package manager binary with environment and PATH setup |
| playwright | `/ov-layers:playwright` | Playwright browser automation (OpenClaw AI snapshots) |
| postgresql | `/ov-layers:postgresql` | PostgreSQL server on port 5432 with pgvector extension |
| pre-commit | `/ov-layers:pre-commit` | Pre-commit git hooks framework via pixi + markdownlint-cli |
| python | `/ov-layers:python` | Python 3.13 runtime via pixi (conda-forge) |
| python-ml | `/ov-layers:python-ml` | ML/AI Python environment (PyTorch, transformers, vLLM, CUDA) |
| qemu-guest-agent | `/ov-layers:qemu-guest-agent` | QEMU guest agent for host-guest communication |
| redis | `/ov-layers:redis` | Redis in-memory data store on port 6379 |
| ripgrep | `/ov-layers:ripgrep` | Fast recursive text search (rg) |
| rocm | `/ov-layers:rocm` | AMD ROCm runtime, OpenCL, and GPU compute support |
| rpmfusion | `/ov-layers:rpmfusion` | RPM Fusion free and nonfree repository configuration |
| rust | `/ov-layers:rust` | Rust compiler and Cargo via system packages (RPM/DEB) |
| sag | `/ov-layers:sag` | ElevenLabs text-to-speech CLI |
| selkies | `/ov-layers:selkies` | Browser-accessible desktop streaming via WebRTC/WebSocket (pixelflux + pcmflux) |
| selkies-desktop | `/ov-layers:selkies-desktop` | Full Selkies streaming desktop metalayer (Chrome, Waybar, automation, accessibility) |
| sherpa-onnx | `/ov-layers:sherpa-onnx` | sherpa-onnx offline text-to-speech |
| socat | `/ov-layers:socat` | Socket relay for port_relay and VM console access |
| songsee | `/ov-layers:songsee` | Audio spectrogram and visualization CLI |
| sqlite | `/ov-layers:sqlite` | SQLite database CLI |
| ssh-client | `/ov-layers:ssh-client` | OpenSSH client tools for SSH agent forwarding |
| sshd | `/ov-layers:sshd` | OpenSSH server and client on port 22 |
| steam | `/ov-layers:steam` | Steam gaming client with gamescope |
| summarize | `/ov-layers:summarize` | Summarize CLI for extracting text/transcripts from URLs and files |
| supervisord | `/ov-layers:supervisord` | Supervisord process manager for multi-service containers |
| sway | `/ov-layers:sway` | Sway Wayland compositor running headless with Mesa GPU drivers |
| sway-desktop | `/ov-layers:sway-desktop` | Sway desktop composition (audio, portals, Chrome, terminal, file manager, status bar) |
| sway-desktop-vnc | `/ov-layers:sway-desktop-vnc` | Sway desktop with VNC remote access via wayvnc on port 5900 |
| swaync | `/ov-layers:swaync` | SwayNotificationCenter for wlroots compositors (sway, labwc) |
| testapi | `/ov-layers:testapi` | FastAPI test service on port 9090 routed via testapi.localhost |
| thunar | `/ov-layers:thunar` | Thunar file manager for Sway desktop environments |
| tmux | `/ov-layers:tmux` | Terminal multiplexer |
| traefik | `/ov-layers:traefik` | Traefik reverse proxy on ports 8000/8080/443 with automatic TLS |
| typst | `/ov-layers:typst` | Typst document processor for typesetting |
| ujust | `/ov-layers:ujust` | Just task runner with ujust wrapper |
| unsloth | `/ov-layers:unsloth` | Unsloth LLM fine-tuning (Tier 1 post-install, no pixi.toml) |
| unsloth-studio | `/ov-layers:unsloth-studio` | Unsloth Studio fine-tuning web UI (Tier 2 meta-layer with pixi.toml) |
| uv | `/ov-layers:uv` | uv Python package manager |
| valkey | `/ov-layers:valkey` | Valkey 9.x key-value store (Redis-compatible) on port 6379 |
| vectorchord | `/ov-layers:vectorchord` | VectorChord PostgreSQL vector similarity extension |
| virtualization | `/ov-layers:virtualization` | QEMU/KVM/libvirt virtualization stack with virt-manager |
| vr-streaming | `/ov-layers:vr-streaming` | OpenXR, OpenVR, and GStreamer libraries for VR streaming |
| vscode | `/ov-layers:vscode` | Visual Studio Code from Microsoft's RPM repository |
| wacli | `/ov-layers:wacli` | WhatsApp CLI for message sending and history sync |
| wf-recorder | `/ov-layers:wf-recorder` | Wayland screen recorder for wlroots compositors |
| waybar | `/ov-layers:waybar` | Waybar status bar and sway-autotile for Sway desktop |
| waybar-labwc | `/ov-layers:waybar-labwc` | Waybar taskbar panel adapted for labwc compositor |
| wayvnc | `/ov-layers:wayvnc` | WayVNC server on port 5900 for Wayland remote access |
| whisper | `/ov-layers:whisper` | OpenAI Whisper local speech-to-text |
| wl-overlay | `/ov-layers:wl-overlay` | Wayland overlay windows via gtk4-layer-shell for recordings |
| wl-screenshot-grim | `/ov-layers:wl-screenshot-grim` | Screenshot via grim (wlr-screencopy) |
| wl-screenshot-pixelflux | `/ov-layers:wl-screenshot-pixelflux` | Screenshot via selkies WebSocket capture bridge |
| wl-record-pixelflux | `/ov-layers:wl-record-pixelflux` | Desktop video recorder via selkies WebSocket capture bridge |
| wl-tools | `/ov-layers:wl-tools` | Compositor-agnostic desktop automation tools (grim, wtype, wlrctl) |
| x11-apps | `/ov-layers:x11-apps` | Desktop applications (terminal, file manager) for X11 containers |
| x11-desktop | `/ov-layers:x11-desktop` | X11 desktop composition (Xorg headless, Openbox, Chrome, terminal, file manager) |
| xdg-portal | `/ov-layers:xdg-portal` | XDG Desktop Portal infrastructure |
| xdg-portal-gnome | `/ov-layers:xdg-portal-gnome` | GNOME XDG Desktop Portal backend (ScreenCast, RemoteDesktop, AT-SPI2) |
| xdg-portal-kde | `/ov-layers:xdg-portal-kde` | KDE XDG Desktop Portal backend (ScreenCast, RemoteDesktop) |
| xdg-portal-niri | `/ov-layers:xdg-portal-niri` | XDG desktop portal integration for Niri compositor (GTK + GNOME backends) |
| xfce4-terminal | `/ov-layers:xfce4-terminal` | Xfce4 terminal emulator for Sway desktop environments |
| xorg-headless | `/ov-layers:xorg-headless` | Xorg X server with dummy video driver and libinput for headless containers |
| xterm | `/ov-layers:xterm` | X11 terminal (XWayland) |
| xurl | `/ov-layers:xurl` | X (Twitter) API CLI for posts, search, DMs, and media |
| yay | `/ov-layers:yay` | AUR helper for Arch Linux (base-devel + yay binary) |

### ov-images (41 skills)

Reference documentation for every enabled Overthink container image.

| Skill | Invocation | Description |
|-------|-----------|-------------|
| arch-ov | `/ov-images:arch-ov` | Arch Linux image with full ov toolchain for container management |
| arch-test | `/ov-images:arch-test` | Arch Linux pacman and AUR package installation test image |
| archlinux | `/ov-images:archlinux` | Base Arch Linux image, root of the pac-based image hierarchy |
| archlinux-builder | `/ov-images:archlinux-builder` | Arch Linux builder with pixi, Node.js, build toolchain, and yay |
| aurora | `/ov-images:aurora` | Aurora DX bootc image with NVIDIA, SSH, ov toolchain, and Go |
| bazzite-ai | `/ov-images:bazzite-ai` | Bazzite NVIDIA bootc image with dev tools, CUDA, Kubernetes, Docker, and desktop apps |
| comfyui | `/ov-images:comfyui` | ComfyUI image generation server with CUDA GPU support |
| debian | `/ov-images:debian` | Base Debian 13 image (disabled, placeholder for Debian-based builds) |
| fedora | `/ov-images:fedora` | Base Fedora 43 image, root of the RPM-based image hierarchy |
| fedora-builder | `/ov-images:fedora-builder` | Builder image with pixi, Node.js, and C/C++ build toolchain |
| fedora-ov | `/ov-images:fedora-ov` | Fedora image with full ov toolchain for container management |
| fedora-nonfree | `/ov-images:fedora-nonfree` | Fedora with RPM Fusion non-free repositories enabled |
| fedora-remote | `/ov-images:fedora-remote` | Fedora image using remote layer references from GitHub |
| fedora-test | `/ov-images:fedora-test` | Test image with Traefik reverse proxy and testapi service |
| githubrunner | `/ov-images:githubrunner` | Self-hosted GitHub Actions runner with full ov toolchain |
| immich | `/ov-images:immich` | Immich photo management server on port 2283 (CPU) |
| immich-ml | `/ov-images:immich-ml` | Immich photo management with CUDA ML backend |
| jupyter | `/ov-images:jupyter` | Lightweight JupyterLab with real-time collaboration (no GPU) |
| jupyter-ml | `/ov-images:jupyter-ml` | Full CUDA ML + JupyterLab + CRDT MCP server |
| jupyter-ml-notebook | `/ov-images:jupyter-ml-notebook` | Full CUDA ML + JupyterLab + CRDT MCP + fine-tuning notebooks |
| nvidia | `/ov-images:nvidia` | NVIDIA GPU base image with CUDA toolkit on Fedora |
| ollama | `/ov-images:ollama` | Standalone Ollama LLM inference server with CUDA GPU support |
| openclaw | `/ov-images:openclaw` | Headless OpenClaw AI gateway on port 18789 |
| openclaw-browser-bootc | `/ov-images:openclaw-browser-bootc` | Bootc VM image with OpenClaw gateway, Chrome, VNC, and PipeWire |
| openclaw-full | `/ov-images:openclaw-full` | Headless OpenClaw with all tool layers (no desktop) |
| openclaw-full-ml | `/ov-images:openclaw-full-ml` | OpenClaw full + ML tools + Ollama + Sway desktop + VNC |
| openclaw-full-sway | `/ov-images:openclaw-full-sway` | OpenClaw with all tools + Sway desktop + VNC (no GPU/ML) |
| openwebui | `/ov-images:openwebui` | Open WebUI on port 8080 with auto-configured LLM providers and MCP |
| openclaw-ollama | `/ov-images:openclaw-ollama` | Headless OpenClaw gateway with local Ollama LLM inference |
| openclaw-ollama-sway-browser | `/ov-images:openclaw-ollama-sway-browser` | Full-stack AI: OpenClaw + all tools + Ollama + Whisper + Sway desktop |
| openclaw-sway-browser | `/ov-images:openclaw-sway-browser` | Maximal OpenClaw with Sway desktop, Chrome, VNC, and all tools |
| python-ml | `/ov-images:python-ml` | GPU-accelerated Python ML environment (CUDA, PyTorch, llama.cpp) |
| selkies-desktop | `/ov-images:selkies-desktop` | Browser-accessible Wayland desktop streamed via Selkies/pixelflux WebSocket |
| selkies-desktop-nvidia | `/ov-images:selkies-desktop-nvidia` | GPU-accelerated selkies-desktop variant with NVIDIA CUDA toolkit |
| sway-browser-vnc | `/ov-images:sway-browser-vnc` | Minimal Sway desktop with VNC remote access and Chrome browser |
| ubuntu | `/ov-images:ubuntu` | Base Ubuntu 24.04 image (disabled, placeholder for Debian-based builds) |
| unsloth-studio | `/ov-images:unsloth-studio` | Unsloth Studio fine-tuning web UI with vLLM on ports 8888/8000 |
| valkey-test | `/ov-images:valkey-test` | Test image with Valkey (Redis-compatible) key-value store |

## Installation

### Via Team Configuration (settings.json)

```json
{
  "enabledPlugins": {
    "ov@ov-plugins": true,
    "ov-dev@ov-plugins": true,
    "ov-jupyter@ov-plugins": true,
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
| Plugins | 5 |
| Skills | 241 |
| Agents | 3 |
| MCP Servers | 2 (GitHub, Jupyter) |
| MCP Tools | 35 (22 GitHub + 13 Jupyter) |
