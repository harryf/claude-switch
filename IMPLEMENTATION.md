# Implementation Guide

How `claude-switch` and `claude-switch-desktop` work under the hood.

---

## Project Structure

```
claude-switch/
├── claude-switch              # Bash script — Claude Code profile switcher
├── claude-switch-desktop      # Bash script — Claude Desktop profile switcher
├── publish.sh                 # Dev tool — copies scripts from ~/bin, checks for PII, commits
├── .pii-patterns              # Grep patterns for personal data (checked before publish)
├── .gitignore                 # Excludes .pii-patterns and internal working docs
├── README.md                  # User-facing documentation
└── IMPLEMENTATION.md          # This file
```

Both scripts are standalone Bash (`#!/usr/bin/env bash`, `set -euo pipefail`). No external dependencies beyond macOS system tools (`security`, `osascript`, `python3`, `pgrep`) and the `claude` CLI.

---

## claude-switch (Claude Code)

### Core Concept

Claude Code uses a single `~/.claude/` directory for all configuration and a single macOS Keychain entry (`"Claude Code-credentials"`) for OAuth authentication. `claude-switch` turns `~/.claude` into a symlink and maintains separate profile directories (`~/.claude-work/`, `~/.claude-personal/`), swapping the symlink target and keychain token on switch.

### Key Functions

#### Token Management

- **`save_current_token(profile_dir)`** — Reads the current keychain token via `security find-generic-password` and writes it to `$profile_dir/.auth-token` (mode 600).
- **`load_token(profile_dir)`** — Reads `$profile_dir/.auth-token`, deletes the current keychain entry, and writes the profile's token via `security add-generic-password`.
- **`restore_keychain_token(token)`** — Directly writes a token string back to the keychain. Used by the trap handler during `--refresh` of a different profile.
- **`get_token_sub_type(token)`** — Extracts `subscriptionType` from the OAuth JSON using `python3`. Used to distinguish work (`team`/`enterprise`) from personal (`pro`/`max`) accounts.

#### Profile Metadata

Each profile directory contains `.profile-meta`, a sourceable shell file:

```bash
PROFILE_EMAIL="user@example.com"
PROFILE_ORG="Acme Corp"
PROFILE_SUB_TYPE="team"
```

- **`introspect_token(token)`** — Runs `claude auth status` (JSON output) to get email and org. **Important caveat:** `claude auth status` reads from the running process's in-memory state, not the keychain. Only reliable immediately after `claude auth login`.
- **`write_profile_meta(dir, email, org, sub_type)`** — Writes the `.profile-meta` file directly from provided values. Used when we already have the data (e.g., during `init`).
- **`update_profile_meta(dir)`** — Combines `introspect_token` + `write_profile_meta`. Only safe to call right after `claude auth login`.

#### Commands

| Command | Function | What It Does |
|---------|----------|-------------|
| `personal`/`work` | `switch_profile()` | Saves current token, swaps symlink, loads target token |
| `status` | `show_status()` | Reads symlink target, keychain token, and `.profile-meta` |
| `init` | `init_cmd()` | 4-step interactive setup: authenticate both accounts, create profile dirs, convert `~/.claude` to symlink |
| `--refresh` | `refresh_cmd()` | Re-authenticates via `claude auth login`, validates token type, persists |
| `--refresh <profile>` | `refresh_other()` | Saves current keychain token, authenticates as other profile, persists, restores original (with trap safety) |
| `--save` | `save_token_cmd()` | Persists whatever is currently in the keychain to the active profile |

#### Init Flow

1. Checks `~/.claude` is a real directory (not already a symlink)
2. Creates timestamped backup: `~/.claude-backup-YYYYMMDDHHMMSS`
3. **Step 1:** Checks for existing keychain token; if valid, offers to reuse it. Otherwise runs `claude auth login`.
4. **Step 2:** Prompts user to switch accounts on claude.ai in their browser.
5. **Step 3:** Runs `claude auth login` again, validates it's a different account (by email).
6. **Step 4:** Moves `~/.claude` → `~/.claude-{name1}`, clones to `~/.claude-{name2}`, clears history/projects on the clone, creates symlink.
7. Profile names are suggested based on `subscriptionType` (`team`/`enterprise` → "work", `pro`/`max` → "personal") but user can override.

#### Switch Flow

1. Check not already on target profile
2. Warn if Claude processes are running (`pgrep -f 'claude'`)
3. Save current keychain token to current profile's `.auth-token`
4. Remove and recreate the `~/.claude` symlink pointing to the target profile directory
5. Load target profile's `.auth-token` into keychain

---

## claude-switch-desktop (Claude Desktop)

### Core Concept

Claude Desktop is an Electron app storing data in `~/Library/Application Support/Claude/`. The total directory is ~8.8GB but only ~5MB is account-specific. `claude-switch-desktop` identifies and swaps only the account-specific files, leaving shared data (Claude.app VM bundles, caches, large binaries) untouched.

### Swapped Files

The script maintains a hardcoded list of 14 files/directories that are account-specific:

```
config.json           — Encrypted OAuth token (Electron safeStorage)
Local Storage/        — LevelDB with account metadata
Cookies               — Session cookies
Cookies-journal       — Cookie write-ahead log
IndexedDB/            — Indexed database
Session Storage/      — Session data
blob_storage/         — Binary blobs
WebStorage/           — Web storage data
local-agent-mode-sessions  — Agent mode session data
shared_proto_db/      — Shared protocol buffers
ant-did/              — Anthropic device ID
DIPS/                 — Bounce tracking data
Trust Tokens/         — Privacy pass tokens
Network Persistent State   — Network state
```

### Profile Snapshots

Profiles are stored as directory snapshots under `~/.claude-desktop-profiles/{work,personal}/`. The active profile is tracked in `~/.claude-desktop-profiles/.active`.

### Key Functions

- **`save_desktop_profile(profile)`** — Copies each swap file from `~/Library/Application Support/Claude/` to the profile snapshot directory.
- **`load_desktop_profile(profile)`** — Copies each swap file from the snapshot back into the Desktop data directory. Validates that `config.json` exists in the snapshot.
- **`quit_claude_desktop()`** — Uses `osascript` to send a quit event to Claude Desktop, then polls `pgrep -xq "Claude"` for up to 5 seconds. Desktop must be quit because it holds file locks on LevelDB and SQLite files.
- **`get_account_display(profile)`** — Reads account info from Claude Code's `.profile-meta` file (not Desktop's own data), so Code profiles must be set up first.

### Commands

| Command | Function | What It Does |
|---------|----------|-------------|
| `personal`/`work` | `switch_profile()` | Quits Desktop, saves current snapshot, loads target snapshot |
| `status` | `show_status()` | Shows active profile, checks for token in `config.json` |
| `init` | `init_cmd()` | 3-step interactive setup: capture first account, sign into second, capture second |
| `--refresh` | `refresh_current()` | Prompts user to re-auth in Desktop, then captures updated snapshot |
| `--refresh <profile>` | `refresh_other()` | Prints manual instructions (can't automate cross-profile Desktop refresh) |
| `--save` | `save_snapshot_cmd()` | Captures current Desktop state to active profile's snapshot |

### Why Electron safeStorage Swapping Works

Desktop encrypts OAuth tokens in `config.json` using Electron's `safeStorage` API. The encryption key is stored in the macOS Keychain as `"Claude Safe Storage"`. Crucially, this key is per-machine, not per-account — so encrypted tokens from either account can be decrypted on the same machine. This is what makes the file-swap approach viable.

---

## publish.sh (Development Workflow)

The publish script automates syncing local script edits into the git repo:

1. **Copy** — Copies `claude-switch` and `claude-switch-desktop` from `~/bin` into the repo directory.
2. **PII Check** — Reads patterns from `.pii-patterns` (one grep pattern per line) and checks the scripts for matches. Aborts if any personal data is found.
3. **Commit** — Shows a `git diff --stat`, prompts for confirmation, then `git add`, `git commit -m "Sync scripts from ~/bin"`, and `git push`.

This workflow assumes development happens on the live scripts in `~/bin` and the repo is the publication target.

---

## Security Considerations

- Auth tokens are stored with `chmod 600` (owner-only read/write)
- Keychain operations use macOS `security` CLI — tokens are protected by the system keychain
- PII check in `publish.sh` prevents accidental publication of personal data (email, username, home directory paths)
- `.pii-patterns` is in `.gitignore` so the patterns themselves aren't published
- The scripts never use `claude auth logout` (which revokes tokens server-side) — they only overwrite keychain entries
- The `--refresh` flow for a different profile uses a bash `trap` to restore the original keychain token if anything fails mid-process

---

## Dependencies

- **macOS** — `security` (Keychain), `osascript` (AppleScript for quitting Desktop), `pgrep`
- **Python 3** — JSON parsing (used via inline `python3 -c` commands)
- **Claude CLI** — `claude auth login`, `claude auth status`
- **Bash 4+** — Arrays, `set -euo pipefail`

---

## Limitations

- **macOS only** — Keychain and Electron safeStorage are macOS-specific. Linux would need a different credential store.
- **Two profiles only** — Profile names are hardcoded to `work` and `personal` in the switch commands (though `init` allows custom names, the `case` statement only matches `personal|work`).
- **Desktop refresh is manual** — Unlike Code (where `claude auth login` is CLI-driven), Desktop auth is interactive inside the app, so cross-profile refresh requires a full switch cycle.
- **`claude auth status` timing** — This command reads from the running process's in-memory state, not the keychain. Profile metadata can only be reliably captured immediately after `claude auth login`. The scripts work around this by using `write_profile_meta` with pre-captured values where possible.
