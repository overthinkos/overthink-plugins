---
name: secrets
description: |
  MUST be invoked before any work involving: ov secrets commands, KeePass .kdbx credential management, credential import/export, or secret database administration.
---

# Secrets - KeePass Credential Management

## Overview

Manage credentials in a KeePass `.kdbx` database. Part of ov's credential store hierarchy — the kdbx backend is used when the system keyring is unavailable (headless servers, SSH sessions) or when explicitly selected via `ov settings set secret_backend kdbx`.

## Credential Store Hierarchy

Resolution order (first match wins):

| Priority | Backend | When used |
|----------|---------|-----------|
| 1 | Environment variable | `OV_VNC_PASSWORD`, etc. |
| 2 | System keyring | GNOME Keyring, KDE Wallet, KeePassXC |
| 3 | KeePass .kdbx | Headless/SSH or `secret_backend: kdbx` |
| 4 | Config file | Plaintext fallback in `config.yml` |

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Create database | `ov secrets init [path]` | Create new .kdbx, auto-set config |
| List entries | `ov secrets list [service]` | List entries, optionally filter by service |
| Get value | `ov secrets get <service> <key>` | Print credential value |
| Set value | `ov secrets set <service> <key> [value]` | Set credential (prompts if no value) |
| Generate value | `ov secrets set <service> <key> --generate` | Generate 32-char hex random value |
| Delete entry | `ov secrets delete <service> <key>` | Remove an entry |
| Import | `ov secrets import [--dry-run]` | Import from config.yml + keyring |
| Export | `ov secrets export [--format yaml\|json]` | Export all entries (plaintext!) |
| Show path | `ov secrets path` | Print resolved .kdbx file path |

## Subcommand Details

### Init

```bash
ov secrets init                          # Default: ~/.config/ov/secrets.kdbx
ov secrets init /path/to/secrets.kdbx    # Custom path
ov secrets init --force                  # Overwrite existing database
```

Creates a new .kdbx database with password confirmation. Automatically sets `secrets.kdbx_path` in config. After init, activate with:

```bash
ov settings set secret_backend kdbx
```

Or it auto-activates when the system keyring is unavailable.

### List

```bash
ov secrets list              # All ov entries
ov secrets list ov/vnc       # Filter by service prefix
```

Prints `service/key` pairs, one per line.

### Get / Set / Delete

```bash
ov secrets get ov/vnc my-image              # Print value
ov secrets set ov/vnc my-image              # Prompt for value securely
ov secrets set ov/vnc my-image mypassword   # Set value inline
ov secrets set ov/vnc my-image --generate   # Generate and print random value
ov secrets delete ov/vnc my-image           # Remove entry
```

The `--generate` flag produces 16 random bytes as 32-character hex string, printed to stdout.

### Import

```bash
ov secrets import --dry-run    # Preview what would be imported
ov secrets import              # Import from config.yml + keyring
```

Collects credentials from:
1. **Config file** — plaintext credentials in `config.yml`
2. **System keyring** — entries tracked in `keyring_keys` config list

Shows source and success/failure for each entry.

### Export

```bash
ov secrets export                  # YAML format (default)
ov secrets export --format json    # JSON format
```

**WARNING:** Exports plaintext credentials. Outputs nested map: `service -> key -> value`.

### Path

```bash
ov secrets path    # Prints resolved path or default location
```

## Common Workflows

### First-Time Setup

```bash
ov secrets init                         # Create database
ov settings set secret_backend kdbx     # Activate kdbx backend
ov secrets import --dry-run             # Preview migration
ov secrets import                       # Migrate existing credentials
```

### Backup / Restore

The `.kdbx` file is a standard KeePass database. You can open it with KeePassXC or any KeePass-compatible tool for backup or manual editing.

## Password Caching

The kdbx master password is cached in the Linux kernel keyring (user keyring, key `ov-kdbx-password`) after the first interactive prompt. All subsequent `ov` commands within the timeout window reuse the cached password automatically -- no re-prompting.

**Password resolution chain:**
1. `OV_KDBX_PASSWORD` environment variable (CI/automation)
2. Linux kernel keyring lookup
3. Interactive prompt (systemd-ask-password or terminal)
4. After prompting, auto-store in kernel keyring with configured TTL

**Configuration:**

```bash
ov settings set secrets.kdbx_cache false       # Disable caching entirely
ov settings set secrets.kdbx_cache_timeout 7200 # Cache for 2 hours instead of 1
```

Default: enabled, 3600 seconds (1 hour) TTL. Uses `keyctl` syscalls via `golang.org/x/sys/unix`. Source: `ov/keyctl.go`.

## Config Keys

| Key | Description |
|-----|-------------|
| `secret_backend` | Force backend: `keyring`, `kdbx`, or `config` |
| `secrets.kdbx_path` | Path to .kdbx file |
| `secrets.kdbx_key_file` | Optional key file for .kdbx |
| `secrets.kdbx_cache` | Enable/disable kernel keyring caching (default: `true`) |
| `secrets.kdbx_cache_timeout` | TTL in seconds for cached password (default: `3600`) |

## Project-Level Secrets (.secrets + direnv)

Separate from ov's credential store, project-level environment variables (e.g., `GMAIL_USER`, `GMAIL_PASSWORD`) are stored in `.secrets` — a GPG-encrypted file at the project root. direnv decrypts it in memory via `ov secrets gpg env` when entering the directory (`eval "$(ov secrets gpg env)"` in `.envrc`).

**This is NOT managed by `ov secrets` (kdbx).** The two systems serve different purposes:

| System | What it manages | How it works |
|--------|----------------|--------------|
| `ov secrets` (kdbx/keyring) | Container-level credentials (VNC passwords, service secrets) | Provisioned at `ov config` time into Podman secrets |
| `ov secrets gpg` + direnv | Project-level shell env vars (API keys, credentials) | GPG-encrypted `.secrets` file, decrypted by direnv on `cd` |

## `ov secrets gpg` — GPG-Encrypted .secrets Management

Manage GPG-encrypted `.secrets` environment files directly from the CLI. All commands shell out to `gpg` (must be in PATH).

### Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Show contents | `ov secrets gpg show [-f FILE]` | Decrypt and print to stdout |
| Export for eval | `ov secrets gpg env [-f FILE]` | Decrypt .secrets as `export KEY='value'` for eval/direnv |
| Edit in editor | `ov secrets gpg edit [-f FILE]` | Decrypt, open `$EDITOR`, re-encrypt |
| Encrypt file | `ov secrets gpg encrypt -r KEY_ID [-i .env] [-o .secrets]` | Encrypt plaintext env file |
| Decrypt file | `ov secrets gpg decrypt [-i .secrets] [-o FILE]` | Decrypt to file or stdout |
| Set a key | `ov secrets gpg set KEY VALUE [-f FILE] [-r KEY_ID]` | Add or update KEY=VALUE |
| Remove a key | `ov secrets gpg unset KEY [-f FILE]` | Remove a key from .secrets |
| Add recipient | `ov secrets gpg add-recipient KEY_ID [-f FILE]` | Re-encrypt with additional recipient |
| List recipients | `ov secrets gpg recipients [-f FILE]` | List GPG key IDs that can decrypt |
| **Import key** | `ov secrets gpg import-key <path>` | Import GPG key from file, directory, or Secret Service |
| **Export key** | `ov secrets gpg export-key [path] [--to-keystore]` | Export GPG key to directory and/or KeePassXC |
| **Setup** | `ov secrets gpg setup` | Configure gpg-agent, import/generate key, store passphrase |
| **Doctor** | `ov secrets gpg doctor [-f FILE]` | Health check: GPG agent, keys, Secret Service, .secrets |

### `ov secrets gpg env`

Decrypt `.secrets` and output `export KEY='value'` lines to stdout. Designed for `eval` or direnv:

```bash
eval "$(ov secrets gpg env)"              # Load secrets into current shell
eval "$(ov secrets gpg env -f .secrets.prod)"  # Load from specific file
```

**Behavior:**
- Silent exit 0 if file doesn't exist (safe for `.envrc` — no error when `.secrets` is absent)
- Parses KEY=VALUE format, skips comments and blank lines, strips surrounding quotes
- Values are single-quoted in output (safe shell escaping via `shellQuote`)
- Replaces the external `dotenv_gpg_if_exists` direnvrc function — no external dependency needed

**Usage in `.envrc`:**
```bash
eval "$(ov secrets gpg env)"
```

### Common Workflows

```bash
# Create .secrets from a plaintext .env file
ov secrets gpg encrypt -r 420DE2B3 -i .env -o .secrets
rm .env

# View contents
ov secrets gpg show

# Add a new secret
ov secrets gpg set API_KEY sk-test-abc123

# Edit in your editor
ov secrets gpg edit

# Remove a secret
ov secrets gpg unset OLD_KEY

# Add another person who can decrypt
ov secrets gpg add-recipient THEIR_KEY_ID

# See who can decrypt
ov secrets gpg recipients
```

### Flags

- `-f, --file` — Path to encrypted file (default: `.secrets` in current directory)
- `-r, --recipient` — GPG key ID (repeatable, required for `encrypt`, optional for `set` on new files)
- `-i, --input` — Input file for encrypt/decrypt (default: `.env` / `.secrets`)
- `-o, --output` — Output file for encrypt/decrypt (default: `.secrets` / stdout)

### `ov secrets gpg import-key` — Import GPG Keys

Import from file, directory, or KeePassXC Secret Service:

```bash
ov secrets gpg import-key ~/Sync/Conf/gpg/           # From directory (all .asc/.gpg + ownertrust.txt)
ov secrets gpg import-key ~/key.asc                   # From single file
ov secrets gpg import-key --from-keystore              # From KeePassXC Secret Service
ov secrets gpg import-key --from-keystore --key-id XX  # Specific key from KeePassXC
ov secrets gpg import-key ~/keys/ --passphrase "xxx"   # With loopback pinentry (no GUI prompt)
```

**Directory import** auto-detects `.asc`, `.gpg`, and `ownertrust.txt` files. **Keystore import** retrieves armored keys stored with schema `org.gnupg.Key` — these are created by `ov secrets gpg export-key --to-keystore`.

The `--passphrase` flag uses GPG's loopback pinentry mode, avoiding the GUI prompt during import.

### `ov secrets gpg export-key` — Export GPG Keys

Export to filesystem and/or KeePassXC:

```bash
ov secrets gpg export-key ~/Sync/Conf/gpg/            # To directory (public.asc, secret.asc, ownertrust.txt)
ov secrets gpg export-key --to-keystore                # To KeePassXC Secret Service
ov secrets gpg export-key ~/backup/ --to-keystore      # Both filesystem and KeePassXC
ov secrets gpg export-key --key-id XXXX --to-keystore  # Specific key
ov secrets gpg export-key --to-keystore --passphrase X # Also store passphrase in Secret Service
```

**Keystore export** stores the armored private key as a Secret Service entry (schema `org.gnupg.Key`, attributes: `keyid`, `uid`). This enables key recovery via `import-key --from-keystore` without filesystem backups.

### `ov secrets gpg setup` — One-Stop GPG Configuration

Configures the full GPG/KeePassXC chain in one command:

```bash
ov secrets gpg setup                                   # Interactive
ov secrets gpg setup --import ~/Sync/Conf/gpg/         # Import key, then configure
ov secrets gpg setup --from-keystore                    # Restore key from KeePassXC
ov secrets gpg setup --passphrase "xxx"                 # Batch mode (no prompts)
```

**Steps performed:**
1. Check prerequisites: `gpg`, `gpg-connect-agent`, `pinentry-qt` (libsecret), `secret-tool`
2. Install `~/.gnupg/gpg-agent.conf` (pinentry-qt, 8h cache, allow-preset-passphrase)
3. Enable systemd socket activation (`gpg-agent.socket`, `gpg-agent-extra.socket`)
4. Restart gpg-agent
5. Import or generate GPG key (RSA-4096, 2-year expiry, git config for name/email)
6. Store passphrase in Secret Service for ALL keygrips (primary + subkeys)
7. Verify encrypt/decrypt round-trip

**Flags:**
- `--import <path>` — Import key from file/directory before setup
- `--from-keystore` — Import key from KeePassXC before setup
- `--passphrase <value>` — Batch mode: generate key + store passphrase without prompts
- `--key-id <id>` — Use specific existing key
- `--skip-secret-service` — Skip passphrase storage in Secret Service

### `ov secrets gpg doctor` — Health Check

Read-only diagnostic of the full GPG/direnv chain:

```bash
ov secrets gpg doctor                    # Check everything
ov secrets gpg doctor -f .secrets.prod   # Check specific file
```

**Checks:** gpg binary, gpg-agent status, gpg-agent.conf (pinentry, cache TTL), systemd sockets, secret keys, Secret Service availability, passphrase storage for each keygrip, key backups in Secret Service, .secrets file recipients and decrypt test.

Returns non-zero exit code if any critical issue found.

## GPG/KeePassXC Integration Architecture

### Passphrase Flow (pinentry-qt ↔ KeePassXC)

```
gpg --decrypt .secrets
  → gpg-agent (needs passphrase for subkey keygrip)
    → spawns pinentry-qt
      → pinentry-qt: secret_password_lookup_nonpageable_sync()
        schema: "org.gnupg.Passphrase", attr: keygrip="<40-char-hex>"
      → KeePassXC returns stored passphrase (if found) → no GUI prompt
      → OR: pinentry-qt shows GUI dialog → user enters passphrase
    → gpg-agent caches passphrase (8h default, 12h max)
  → decryption succeeds
```

**Key facts:**
- pinentry-qt links against libsecret and queries KeePassXC via `org.freedesktop.secrets` D-Bus API
- Passphrase lookup uses schema `org.gnupg.Passphrase` with attribute `keygrip`
- Each GPG key has multiple keygrips (one per key/subkey) — passphrases must be stored for ALL keygrips
- `ov secrets gpg setup` stores passphrases for all keygrips automatically
- gpg-agent's in-memory cache (8h/12h) reduces Secret Service lookups

### Key Storage in KeePassXC

Two Secret Service schemas:
- **`org.gnupg.Passphrase`** — passphrase per keygrip (compatible with pinentry-qt auto-lookup)
- **`org.gnupg.Key`** — armored private key backup (for `import-key --from-keystore` recovery)

Both are standard Secret Service entries visible in KeePassXC's FdoSecrets-exposed group.

### Error Diagnostics

When decryption fails, `ov secrets gpg` now prints actionable diagnostics instead of raw GPG errors:
- Which key ID the file is encrypted for
- Whether the key exists in the local keyring
- Whether the key is backed up in Secret Service (with restore command)
- Agent status, config status, Secret Service availability
- Specific `ov secrets gpg` commands to fix each issue

## Key Management Workflows

### First-Time Setup (New Machine)

```bash
# Option A: Import existing key from backup
ov secrets gpg setup --import ~/Sync/Conf/gpg/

# Option B: Restore from KeePassXC (if key was previously exported)
ov secrets gpg setup --from-keystore

# Option C: Generate fresh key
ov secrets gpg setup
# Then re-encrypt .secrets: ov secrets gpg encrypt -r <NEW_KEY_ID> -i .env -o .secrets
```

### Backup Key to KeePassXC

```bash
ov secrets gpg export-key --to-keystore              # Key backup
ov secrets gpg export-key ~/Sync/Conf/gpg/           # Filesystem backup too
```

### Restore Key on New Machine

```bash
ov secrets gpg import-key --from-keystore            # From KeePassXC
ov secrets gpg setup                                 # Configure agent + store passphrase
```

### Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `gpg: No secret key` | Key not in keyring | `ov secrets gpg import-key <path>` |
| Constant passphrase prompts | Passphrase not in Secret Service | `ov secrets gpg setup` (stores for all keygrips) |
| `decrypting .secrets failed` | Diagnostics printed automatically | Follow suggested `ov secrets gpg` commands |
| `Secret Service not available` | KeePassXC not running/unlocked | Start KeePassXC, unlock database |
| `secret-tool store` fails | No FdoSecrets exposed group | KeePassXC → right-click group → "Mark as Secret Service exposed" |
| Passphrase stored but still prompted | Stored for wrong keygrip | `ov secrets gpg doctor` shows which keygrips need passphrases |

### Using .secrets Inside Containers

For GPG agent forwarding into containers (so `gpg --decrypt` works inside), use the `agent-forwarding` layer. See `/ov-layers:agent-forwarding` for details. The container's GPG uses the host's agent via a forwarded socket — no GPG agent runs inside the container.

## Cross-References

- `/ov:config` — `secret_backend`, `secrets.kdbx_path` settings keys
- `/ov:service` — container secrets (`secrets` field in layer.yml, provisioned at `ov config`)
- `/ov-layers:agent-forwarding` — SSH/GPG agent forwarding into containers
- `/ov-layers:gnupg` — GnuPG package layer
- `/ov-layers:openwebui` — two-tier secrets pattern: podman secrets (`WEBUI_SECRET_KEY` auto-generated) + GPG `.secrets` (API keys)
- `/ov-layers:direnv` — direnv environment loader

## Source

`ov/secrets_cmd.go` (CLI commands), `ov/secrets_gpg.go` (GPG .secrets commands, key management, diagnostics), `ov/credential_kdbx.go` (KdbxStore backend).

## When to Use This Skill

**MUST be invoked** when the task involves `ov secrets` commands, KeePass .kdbx credential management, GPG-encrypted `.secrets` file management, GPG key management, Secret Service integration, credential import/export, or secret database administration. Invoke this skill BEFORE reading source code or launching Explore agents.
