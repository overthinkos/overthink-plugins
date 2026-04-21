---
name: container-nesting
description: |
  Rootless nested podman/buildah/skopeo recipe. Ships zero cap_add — works
  via surgical `unmask=/proc/*` security_opt plus dual-location
  containers.conf/storage.conf/policy.json plus two canonical env vars
  plus subuid layout that fits inside the outer user namespace. Authoritative
  source for the `mount_too_revealing()` kernel RCA.
  Use when working with nested containers, the container-nesting layer,
  or any "rootless-in-rootless podman" question.
---

# container-nesting -- Rootless nested podman, buildah, skopeo

## Overview

Adds everything needed to run **rootless** podman/buildah/skopeo **inside
a rootless outer container** — at the default uid 1000, with zero added
capabilities, no `--privileged`, no `seccomp=unconfined`, no
`label=disable`. The recipe is a direct port of `quay.io/podman/stable`'s
canonical configuration, ported into the ov layer system so any image
can compose it.

## Layer Properties

| Property | Value |
|----------|-------|
| `cap_add` | **(none)** |
| `security_opt` | `unmask=/proc/*` |
| `devices` | `/dev/fuse`, `/dev/net/tun` |
| Volumes | `storage` at `/var/lib/containers/storage` (only used by root images) |
| Env | `OV_BUILD_ENGINE=podman`, `OV_RUN_ENGINE=podman`, `_CONTAINERS_USERNS_CONFIGURED=""`, `BUILDAH_ISOLATION=chroot` |

## Packages

**RPM:** `buildah`, `fuse-overlayfs`, `shadow-utils`, `skopeo`,
`tailscale`, `libsecret` (Tailscale from the `tailscale-stable` repo).

**Pacman:** `buildah`, `crun`, `fuse-overlayfs`, `libsecret`, `podman`,
`shadow`, `skopeo`, `tailscale`.

**Arch correction (current):** the pac list previously held `docker`
instead of `podman` and omitted `crun`. Neither was right — the layer's
whole purpose is **rootless nested podman**, and the `containers.conf`
shipped by this layer explicitly sets `runtime = "crun"`. The correction
was caught when `ov mcp serve`'s `status` tool failed inside `arch-ov`
with `executable file "podman" not found in $PATH`, because the fedora
pacman population incorrectly pulled `docker` and no podman binary ever
made it into the Arch-based image. Rpm users have always gotten
`podman` transitively via the Fedora base image; Arch has no such
transitive pull, so both `podman` and `crun` must be declared
explicitly.

## The kernel-level RCA (why none of the obvious fixes work)

This is the load-bearing section — if you're here because "nested
podman fails with `crun: mount proc to proc: Operation not permitted`",
read this before trying anything else.

### What actually fails

When the inner podman starts a new container (e.g., alpine) inside a
rootless outer container, `crun` tries to mount a fresh procfs for the
new container's mount namespace. The call is roughly:

```c
mount("proc", "/proc", "proc", MS_NOSUID|MS_NODEV|MS_NOEXEC, NULL);
```

The Linux kernel refuses with `EPERM`. Not because of capabilities,
not because of seccomp, not because of SELinux.

### Why the kernel refuses

`fs/namespace.c:mount_too_revealing()` is a security check introduced
to prevent information leakage across user-namespace boundaries. It
rejects a procfs mount when:

- The calling process is in a **descendant user namespace** of the
  existing procfs's owning user namespace, AND
- The existing procfs has **submounts** that the new mount would
  expose (the classic example: `/proc/kcore` has been bind-mounted
  over with `/dev/null` to hide kernel memory, and a fresh mount
  would "un-hide" it).

Podman's default rootless outer container generates an OCI spec with
`linux.maskedPaths` covering:

```
/sys/kernel, /proc/acpi, /proc/kcore, /proc/keys,
/proc/latency_stats, /proc/sched_debug, /proc/scsi,
/proc/timer_list, /proc/timer_stats,
/sys/devices/virtual/powercap, /sys/firmware,
/sys/fs/selinux, /proc/interrupts
```

Each of these paths is either bind-mounted over with `/dev/null` or
mounted as a read-only tmpfs. When the inner container tries to mount
its own fresh `/proc`, the kernel sees that would reveal those paths
→ `mount_too_revealing` → `EPERM`.

### Why capability-based fixes don't work

Empirically tested on 2026-04-19:

| Attempt | Result |
|---|---|
| `--cap-add=SYS_ADMIN` | FAIL — caps aren't the issue |
| `--cap-add=ALL` | FAIL — same |
| `--cap-add=ALL --security-opt seccomp=unconfined --security-opt label=disable` | FAIL — caps + seccomp + SELinux aren't the issue |
| `--privileged` | PASS — but only because `--privileged` coincidentally also removes the masked_paths |
| `--security-opt unmask=/proc/*` | **PASS** — surgical fix, no caps needed |

`unmask=/proc/*` tells podman NOT to emit those `maskedPaths` entries
for /proc on the outer container. With nothing to mismatch,
`mount_too_revealing` has nothing to reject. The inner `/proc` mount
proceeds cleanly.

This layer's `security:` block is `security_opt: [unmask=/proc/*]` +
`devices: [/dev/fuse, /dev/net/tun]`. No capability added. No seccomp
touched. No SELinux touched. The surgical minimum.

### Security trade-off

`unmask=/proc/*` exposes `/proc/kcore` (kernel memory) and
`/proc/keys` (kernel keyring) on the outer container's filesystem.

Reading those files still requires `CAP_SYS_ADMIN` **in the init user
namespace** — which a rootless container never has. The actual
information leak is minimal. Compared to `--privileged` (which
ALSO removes the masks, plus grants every capability, plus disables
seccomp, plus passes through every host device, plus disables path
masking entirely), this is the least-privilege fix available.

## Subuid / subgid layout (must fit inside the outer namespace)

`ov shell` launches the outer container with `--userns=keep-id:uid=1000,gid=1000`
(default — see `ov/shell.go:254`). That creates a uid_map inside the
outer of:

```
0    1000    1           # inner uid 0 → host uid 1000
1    100000  65535       # inner uid 1-65535 → host uid 100000-165534
```

So inside the outer, **only inner uids 0-65535 exist**. Subid
delegation ranges that fall outside this window fail at
`newuidmap write to uid_map: EPERM`.

The layer emits two non-overlapping ranges for the primary uid-1000
user, skipping uid 1000 itself (because keep-id already owns it):

```
user:1:999
user:1001:64535
```

…plus a full-range entry for root (used by `fedora-ov`/`arch-ov`/
`githubrunner`, which run as uid 0):

```
root:1:65535
```

This pattern matches `quay.io/podman/stable`'s `/etc/subuid` layout
exactly. Older revisions of this layer used `524288:65536` — that range
falls outside the outer namespace's mapped window and was the cause of
an obscure `newuidmap` write failure during the 2026-04-19 RCA.

The `newuidmap`/`newgidmap` binaries get `cap_setuid=ep` / `cap_setgid=ep`
file capabilities (via a dedicated task) so any uid invoking them can
delegate subids.

## Config files — written to both system-wide and user locations

Rootless podman **prefers `~/.config/containers/*` over `/etc/containers/*`**.
Writing only the system-wide location is a no-op for the desktop user.
This layer writes every config to both locations.

### `containers.conf`

Identical at `/etc/containers/containers.conf` and `~/.config/containers/containers.conf`:

```ini
[containers]
cgroups = "disabled"
cgroupns = "host"
ipcns = "host"
netns = "host"
userns = "host"
utsns = "host"
log_driver = "k8s-file"

[engine]
cgroup_manager = "cgroupfs"
events_logger = "file"
runtime = "crun"
```

Why each setting:

- `cgroups = "disabled"` — rootless cgroupv2 delegation isn't
  guaranteed at nesting depth 2+; disabling avoids "cannot set up
  cgroup" errors.
- `cgroup_manager = "cgroupfs"` — systemd cgroup delegation likewise
  not guaranteed.
- `netns = "host"` — pasta (rootless networking) needs
  `/proc/sys/net/ipv4/ping_group_range` writable, which is
  read-only in a rootless outer. `netns=host` makes the inner reuse
  the outer's netns, bypassing the need.
- `userns = "host"` — required by the `mount_too_revealing` analysis
  above. Without it, the inner podman creates a descendant userns and
  hits the kernel check on every `/proc` mount.
- `ipcns`, `utsns`, `cgroupns = "host"` — match the canonical
  `quay.io/podman/stable` config; prevents a cascade of namespace
  permission errors observed when only some are set.

### `storage.conf`

System-wide (for root images) at `/etc/containers/storage.conf`:

```ini
[storage]
driver = "overlay"
runroot = "/run/containers/storage"
graphroot = "/var/lib/containers/storage"

[storage.options.overlay]
mount_program = "/usr/bin/fuse-overlayfs"
mountopt = "nodev,fsync=0"
```

User-level (for uid-1000 images) at `~/.config/containers/storage.conf` —
**different graphroot** so it's user-writable:

```ini
[storage]
driver = "overlay"
runroot = "${HOME}/.local/share/containers/run"
graphroot = "${HOME}/.local/share/containers/storage"

[storage.options.overlay]
mount_program = "/usr/bin/fuse-overlayfs"
mountopt = "nodev,fsync=0"
```

`mount_program = "/usr/bin/fuse-overlayfs"` is the critical line —
the kernel overlay driver can't mount from a rootless outer (no
`CAP_SYS_ADMIN` in init userns); `fuse-overlayfs` can.

If the user-level file is missing, rootless podman uses the system
default, which points at `/var/lib/containers/storage` — unwritable by
uid 1000 → `mkdir graphroot: permission denied` on first `podman run`.

### `policy.json`

Same content at both `/etc/containers/policy.json` and
`~/.config/containers/policy.json`:

```json
{"default":[{"type":"insecureAcceptAnything"}]}
```

Without this, `podman pull` fails with `no policy.json file found`.

## Env vars — the two hidden contracts

| Env var | Value | Role |
|---|---|---|
| `_CONTAINERS_USERNS_CONFIGURED` | `""` (empty string, SET not UNSET) | Tells the inner podman "you're already inside a rootless user namespace". Without this, the inner re-execs itself via `newuidmap` to create a new descendant user namespace — defeating `userns=host` in containers.conf and re-triggering `mount_too_revealing`. |
| `BUILDAH_ISOLATION` | `chroot` | Tells buildah RUN steps to use chroot isolation instead of the OCI runtime. Without this, nested `podman build` falls back to OCI isolation which creates a descendant user namespace and hits the same kernel check. |

Both are baked into the layer's `env:` section so they land in the
OCI env of any image composing this layer.

## Image-level compatibility (union semantics)

`ov/security.go:66-97` **unions** image-level `CapAdd`, `SecurityOpt`,
`Devices` onto the layer-level merged set (via `appendUnique`). Image
values can only ADD, never strip.

Consequence: images that want the old full-hammer posture
(`fedora-ov`, `arch-ov`, `githubrunner`) must assert it at the image
level, not expect this layer to donate it. Their `image.yml` entries
carry:

```yaml
security:
  cap_add: [ALL]
  security_opt:
    - label=disable
    - seccomp=unconfined
```

The resolved OCI label then unions to
`cap_add:[ALL] + security_opt:[unmask=/proc/*, label=disable, seccomp=unconfined]`,
which matches their historical posture.

Rootless images like `/ov-images:selkies-desktop-ov` don't add an
image-level `security:` block, so the resolved posture stays at
`security_opt:[unmask=/proc/*]` only — zero capability escalation.

## Cross-distro coverage

`rpm:` (Fedora), `pac:` (Arch), plus compound tag `debian,ubuntu:` for shared packages (`podman`, `buildah`, `skopeo`, `fuse-overlayfs`, `crun`, `uidmap`, `passwd`, `libsecret-1-0`). Each of `debian:13:` and `ubuntu:24.04:` additionally declares the Tailscale apt repo (`https://pkgs.tailscale.com/stable/debian` or `.../ubuntu`) with signed-by key for the `tailscale` package. Drops on deb: none at the nesting recipe level — all the critical pieces (containers.conf, storage.conf, subuid layout, `_CONTAINERS_USERNS_CONFIGURED=""` env) are format-agnostic task content.

## Usage — rootless image (uid 1000)

```yaml
# image.yml
selkies-desktop-ov:
  base: nvidia
  layers:
    - selkies-desktop
    - ov-full
    - container-nesting   # donates unmask + devices + config + env
    - ...
  # NO uid/gid/user/network override
```

## Usage — root image (uid 0)

```yaml
# image.yml
fedora-ov:
  base: fedora
  uid: 0
  gid: 0
  user: root
  network: host
  security:
    cap_add: [ALL]
    security_opt:
      - label=disable
      - seccomp=unconfined
  layers:
    - ov-full
    - container-nesting
    - ...
```

Both paths work; they just resolve to different OCI security labels.

## Verification

```bash
# Rootless posture on selkies-desktop-ov
ov image inspect selkies-desktop-ov | jq '.HostConfig? // .Config.Labels."org.overthinkos.security"'
# → cap_add:[], security_opt:[unmask=/proc/*], devices:[/dev/fuse,/dev/net/tun]

# Nested podman smoke (inside the running container)
ov shell selkies-desktop-ov -c 'podman run --rm quay.io/libpod/alpine:latest true'
# → exit 0, NO "mount proc to proc: Operation not permitted"

# Diagnostic: inspect the OCI spec generated for a nested container
ov shell selkies-desktop-ov -c '
  podman create --name t quay.io/libpod/alpine:latest /bin/true >/dev/null
  sf=$(find ~/.local/share/containers -name config.json -path "*/userdata/*" | head -1)
  jq ".linux.maskedPaths" "$sf"'
# → empty list or no /proc entries = unmask worked
```

If `mount proc to proc: EPERM` still happens after a rebuild, check in
this order:

1. `env | grep _CONTAINERS_USERNS_CONFIGURED` — must print one line with empty value (SET, not UNSET).
2. `grep '^userns' /etc/containers/containers.conf ~/.config/containers/containers.conf` — both must say `host`.
3. `cat /etc/subuid` — must show the 1:999 + 1001:64535 pattern for the primary user.
4. `podman inspect <outer-container> --format '{{.HostConfig.SecurityOpt}}'` — must include `unmask=/proc/*`.

## Used In Images

- `/ov-images:selkies-desktop-ov` — rootless path; image-level adds nothing
- `/ov-images:fedora-ov` — root path; image-level adds `cap_add:[ALL] + security_opt:[label=disable, seccomp=unconfined]`
- `/ov-images:arch-ov` — same root path as fedora-ov
- `/ov-images:githubrunner` — same root path; doesn't compose the full ov toolchain but keeps nested podman for CI workloads

## Related Layers

- `/ov-layers:ov-full` — pairs with container-nesting in ov-toolchain images (adds the `ov` binary + VM + encrypted storage tools)
- `/ov-layers:virtualization` — supervisord-managed rootless libvirt (`virtqemud`, `virtnetworkd`). Pairs with container-nesting for images that need both nested containers AND nested VMs
- `/ov-layers:sshd` — sibling enabling remote access to nested-container hosts

## Related Commands

- `/ov:build` — build images that ship nested podman
- `/ov:shell` — run nested podman/buildah commands inside the outer
- `/ov:generate` — Containerfile generation (the `service:` supervisord fragments for container-nesting consumers go through the fragment_assembly init model)
- `/ov:config` — image-level `security:` union with layer-level when deploying

## When to Use This Skill

**MUST be invoked** when:

- Authoring or debugging any layer/image that needs nested podman, buildah, or skopeo.
- Chasing `mount_too_revealing` / `mount proc to proc: Operation not permitted` errors — this is the authoritative RCA.
- Choosing between `--privileged`, `cap_add: ALL`, and `unmask=/proc/*` — this skill documents why the surgical `unmask` fix is the minimum-privilege path and the others are hammers.
- Evaluating the security posture of `/ov-images:selkies-desktop-ov`, `/ov-images:fedora-ov`, `/ov-images:arch-ov`, or `/ov-images:githubrunner`.

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov image test`, `ov test`)
