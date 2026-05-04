---
name: keepassxc-keyring
description: |
  Configure KeePassXC as the freedesktop.org Secret Service provider on a
  target:local host: enable FdoSecrets, autostart KeePassXC, disable
  competing daemons (gnome-keyring + kwallet) at the per-user XDG-autostart
  and systemd-user-unit layers, install pinentry/libsecret/keyutils, and
  install generic direnv shell hooks for bash/zsh/fish.
  Use when adding KeePassXC as the Secret Service backend on a host (NOT
  for adding the binary to a container image — use /ov-foundation:keepassxc
  for that).
---

# keepassxc-keyring -- KeePassXC as the system Secret Service

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `keepassxc`, `gnupg` |
| Ports | none |
| Service | none (KeePassXC autostarted via XDG `~/.config/autostart/keepassxc.desktop`, not a systemd unit) |
| Install files | `tasks:` only |
| Target | `target: local` (host deploys); harmless on container images but pointless |

## Packages

- Arch (`pac:`): `pinentry`, `libsecret`, `keyutils`
- Fedora (`rpm:`): `pinentry-qt`, `libsecret`, `keyutils`
- Debian/Ubuntu (`deb:`): `pinentry-qt`, `libsecret-tools`, `keyutils`

`pinentry` provides the GUI passphrase dialog that gpg-agent spawns; `libsecret` provides `secret-tool` and the C client lib that pinentry-qt uses to query KeePassXC; `keyutils` provides `keyctl` for the kernel-keyring caching of `ov`'s kdbx-master-password.

## What this layer changes on the host

1. `~/.config/keepassxc/keepassxc.ini` — writes `[FdoSecrets] Enabled=true` block (and three plugin-behavior toggles).
2. `~/.config/autostart/keepassxc.desktop` — XDG autostart entry that launches KeePassXC on every session login.
3. `~/.config/autostart/<competitor>.desktop` — for each known competing Secret Service daemon (`gnome-keyring-secrets`, `gnome-keyring-ssh`, `gnome-keyring-pkcs11`, `org.kde.kwalletd[5,6]`, `kwalletd[5,6]`), writes an override with `Hidden=true` + `X-GNOME-Autostart-enabled=false`. Per the XDG spec, user-level overrides take precedence over `/etc/xdg/autostart/`, so the system package stays untouched and the change is fully reversible by `ov deploy del`.
4. `systemctl --user disable --now <unit>` for the same competitors' systemd user units (`gnome-keyring-daemon.socket`, `gnome-keyring-daemon.service`, the kwallet service variants). Idempotent, silent if a unit doesn't exist.
5. `~/.gnupg/gpg-agent.conf` — `pinentry-program /usr/bin/pinentry-qt` (the libsecret-linked pinentry that talks to KeePassXC for GPG passphrase storage).
6. `~/.config/environment.d/ssh-agent.conf` — exports `SSH_AUTH_SOCK="${XDG_RUNTIME_DIR}/ssh-agent.socket"` for systemd-bootstrapped sessions.
7. **Per-shell init exports (post-2026-05 cutover, via the `shell:` schema):** for non-systemd-bootstrapped shells (tmux from a screen-locked session, ssh-in shells without a fresh login, scripts), the layer's `shell:` block lands managed-block exports of `SSH_AUTH_SOCK` (guarded with `command -v` / socket-existence check), `KEEPASSXC_DATABASE` advisory pointer, and `GPG_TTY=$(tty)`. bash/zsh/sh share one POSIX-style snippet; fish gets a syntactically-correct counterpart via `set -gx`. environment.d (item 6) and the `shell:` block coexist with no conflict — environment.d wins under systemd, the shell-rc lines fill the gap when systemd isn't in the loop.
8. **Direnv shell hooks come from the `direnv` layer** (declared via `requires: [direnv]`). The pre-2026-05 inline `cmd:` heredocs that wrote `direnv-hook` fenced blocks have been DELETED in the same hard-cutover PR — the responsibility moved to where it belongs (the direnv layer's own `shell:` block). On hosts that previously had the legacy blocks, `ov migrate shell-schema` cleans them up.
9. **systemd user service for KeePassXC** with `Restart=on-failure` and explicit dependency on `graphical-session.target`.

## What this layer DOES NOT do

- **Marking a `.kdbx` group as "Secret Service exposed"** — this is per-database state inside the `.kdbx` file itself, not in `keepassxc.ini`. It is a one-time GUI action (right-click group → "Mark as Secret Service exposed") performed by the user once per database.
- **`ov secrets gpg setup`** — the gpg-agent.conf, systemd socket, and Secret-Service passphrase storage are still wired by the user-invoked `ov secrets gpg setup` command. This layer just installs the packages that command requires (`pinentry-qt`, `libsecret`, `keyutils`).
- **Removing or downgrading** any system package. All competitor disabling is per-user, reversible.

## Usage

```yaml
# local.yml
local:
  cachyos-dx:
    layers:
      - direnv
      - gnupg
      - keepassxc
      - keepassxc-keyring     # this layer
      ...
```

Order with respect to `direnv`/`gnupg`/`keepassxc` doesn't strictly matter (the layer system topo-sorts via `depends:`), but listing them adjacently makes the YAML self-documenting.

## Used In Images / Templates

- `cachyos-dx` (kind:local template, applied to the `qc` deployment)

## Tests

Build-scope (run on package install):

- `keepassxc-binary` — `/usr/bin/keepassxc` exists.
- `pinentry-qt-installed` — at least one of `pinentry-qt`, `pinentry-qt5`, `pinentry-qt6` resolvable on PATH.
- `secret-tool-installed` — `secret-tool` resolvable on PATH.
- `keyutils-installed` — `keyctl` resolvable on PATH.

Deploy-scope (run on the host post-`ov deploy add` against the running user's HOME):

- `keepassxc-ini-fdosecrets` — `[FdoSecrets] Enabled=true` written.
- `keepassxc-autostart` — autostart .desktop file readable.
- `gnome-keyring-secrets-disabled` — competitor override has `Hidden=true` (or the file doesn't exist on this host, which is also fine).
- `direnv-fish-hook` — fish hook file contains `direnv hook fish`.
- `direnv-bash-hook` — `~/.bashrc` contains the bash hook.

## Relationship to other skills

| Skill | Relationship |
|-------|--------------|
| `/ov-foundation:keepassxc` | The package-only layer. `keepassxc-keyring` `depends:` on it; never composes it. |
| `/ov-foundation:gnupg` | Same — keepassxc-keyring `depends:` on gnupg, never composes. |
| `/ov-foundation:agent-forwarding` | Distinct concern. agent-forwarding is for FORWARDING the host's GPG/SSH agents INTO containers via socket bind-mounts. keepassxc-keyring is about turning the host itself into a Secret Service server. Both can be active simultaneously. |
| `/ov-build:secrets` | The CLI surface that talks to KeePassXC after this layer is in place. `ov secrets gpg setup` and `ov secrets gpg doctor` find pinentry-qt + libsecret + keyutils on PATH because this layer installed them. |
| `/ov-coder:direnv` | Installs the direnv binary. This layer adds the missing piece (the shell hook) for `.envrc` to actually trigger on `cd`. |
| `/ov-build:layer` | Layer authoring reference. |
| `/ov-build:eval` | `eval:` block format reference. |

## Why a separate layer (and not edits to `keepassxc` or `agent-forwarding`)

- The `keepassxc` layer is consumed by container images (`/ov-selkies:selkies-desktop-bootc`) where FdoSecrets and autostart make no sense.
- `agent-forwarding` is a clean metalayer (`gnupg + direnv + ssh-client`) used by 27 application images. Adding host-only behavior there would polute every container build that composes it.
- Keeping host-only secret-service wiring in its own layer means containers stay containers and hosts stay hosts.

## When to Use This Skill

Use when the user asks about:

- Making KeePassXC the default Secret Service on a host.
- The `.envrc` + `.secrets` workflow not triggering on `cd` (likely missing direnv hook from this layer).
- gnome-keyring or kwallet conflicting with KeePassXC over the `org.freedesktop.secrets` D-Bus name.
- Why ov's keyring backend "can't find" the user's KeePassXC database (FdoSecrets plugin disabled in keepassxc.ini).
- The `cachyos-dx` template's keyring/direnv/gpg block.

## Related

- `/ov-foundation:keepassxc` — package install (parent dependency)
- `/ov-foundation:gnupg` — gpg binary (parent dependency)
- `/ov-foundation:agent-forwarding` — sibling: container-side agent forwarding metalayer
- `/ov-build:secrets` — `ov secrets` and `ov secrets gpg` CLI surface
- `/ov-coder:direnv` — direnv binary layer (the shell hook this layer adds completes the chain)
- `/ov-build:layer` — layer authoring reference
- `/ov-build:eval` — declarative testing reference
- `/ov-build:local-spec` — `kind: local` template reference (where this layer is composed)
- `/ov-advanced:local-deploy` — `target: local` deploy semantics
