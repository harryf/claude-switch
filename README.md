# claude-switch

Switch between two Anthropic accounts (e.g., work Team/Enterprise + personal Pro/Max) on a single macOS machine, with one-command switching for both **Claude Code** (CLI) and **Claude Desktop**.

**Platform:** macOS (uses macOS Keychain; Linux would need adaptation)

---

## TL;DR — Quick Install

```bash
# Clone
git clone git@github.com:harryf/claude-switch.git
cd claude-switch

# Add to PATH
cp claude-switch claude-switch-desktop /usr/local/bin/
chmod +x /usr/local/bin/claude-switch /usr/local/bin/claude-switch-desktop

# Set up Claude Code switching (interactive — authenticates both accounts)
claude-switch init

# Set up Claude Desktop switching (interactive — captures both profiles)
claude-switch-desktop init

# Switch!
claude-switch work              # Claude Code → work account
claude-switch personal          # Claude Code → personal account
claude-switch-desktop work      # Claude Desktop → work account
claude-switch-desktop personal  # Claude Desktop → personal account
```

> **WARNING: Back up your Claude configuration first!** While `claude-switch init` creates a timestamped backup of `~/.claude/` automatically, you may want to make your own backup before running it. This tool modifies your `~/.claude/` directory structure, macOS Keychain entries, and Claude Desktop application data. **The authors accept no responsibility for any damage to your Claude installation, lost configuration, or corrupted auth tokens.** Use at your own risk.

---

## How It Works

### Claude Code

Claude Code stores a single OAuth token in the macOS Keychain (`"Claude Code-credentials"`). All configuration, hooks, history, and project memory lives under `~/.claude/`.

`claude-switch` manages two profile directories (`~/.claude-work/` and `~/.claude-personal/`) and makes `~/.claude` a symlink pointing to the active one. On switch, it saves the current keychain token, swaps the symlink, and loads the target profile's token.

### Claude Desktop

Claude Desktop is an Electron app that stores auth across multiple files inside `~/Library/Application Support/Claude/` (~8.8GB total, but only ~5MB is account-specific). `claude-switch-desktop` swaps just the account-specific files while leaving shared data (VM bundles, caches, binaries) in place.

---

## Installation

### Option A: Copy to system PATH

```bash
git clone git@github.com:harryf/claude-switch.git
cd claude-switch
cp claude-switch claude-switch-desktop /usr/local/bin/
chmod +x /usr/local/bin/claude-switch /usr/local/bin/claude-switch-desktop
```

### Option B: Symlink from the cloned repo

```bash
git clone git@github.com:harryf/claude-switch.git
cd claude-switch
ln -s "$(pwd)/claude-switch" ~/bin/claude-switch
ln -s "$(pwd)/claude-switch-desktop" ~/bin/claude-switch-desktop
```

Make sure `~/bin` is on your `PATH`.

---

## Setup

### Claude Code — `claude-switch init`

```bash
claude-switch init
```

This interactive setup will:
1. Back up `~/.claude/` to a timestamped directory
2. Authenticate both accounts via `claude auth login` (opens browser)
3. Detect each account's email, org, and subscription type
4. Create profile directories (`~/.claude-work/`, `~/.claude-personal/`)
5. Convert `~/.claude` to a symlink pointing to the active profile
6. Store tokens and account metadata per profile

### Claude Desktop — `claude-switch-desktop init`

```bash
claude-switch-desktop init
```

This interactive setup will:
1. Ask you to sign into your first account in Claude Desktop
2. Capture a snapshot of the account-specific files
3. Ask you to sign out and sign into your second account
4. Capture that snapshot too
5. Set the active Desktop profile

> **Note:** Claude Code profiles (`claude-switch init`) must be set up first — Desktop uses the Code profile metadata to display account info.

---

## Usage

### Claude Code

```bash
claude-switch status              # Show active profile and account info
claude-switch work                # Switch to work profile
claude-switch personal            # Switch to personal profile
claude-switch --refresh           # Re-login current profile, persist new token
claude-switch --refresh work      # Re-login work profile while staying on personal
claude-switch --save              # Persist current keychain token to active profile
claude-switch init                # First-time interactive setup
```

After switching, restart any running Claude Code sessions.

### Claude Desktop

```bash
claude-switch-desktop status      # Show active profile
claude-switch-desktop work        # Switch to work profile (quits Desktop first)
claude-switch-desktop personal    # Switch to personal profile
claude-switch-desktop --refresh   # Re-login current Desktop profile, update snapshot
claude-switch-desktop --save      # Capture current Desktop state to profile snapshot
claude-switch-desktop init        # First-time interactive setup
```

After switching, launch Claude Desktop with `open -a Claude`.

---

## How Auth Tokens Work

### Claude Code

The keychain token is JSON containing OAuth credentials:

```json
{
  "claudeAiOauth": {
    "accessToken": "sk-ant-oat01-...",
    "refreshToken": "sk-ant-ort01-...",
    "expiresAt": 1772984739060,
    "scopes": ["user:inference", "user:profile", "..."],
    "subscriptionType": "team",
    "rateLimitTier": "default_claude_max_5x"
  }
}
```

The token does **not** contain your email — Claude Code resolves that via API. The `subscriptionType` field distinguishes accounts (`team`/`enterprise` for work, `pro`/`max` for personal).

Each profile stores its token in `.auth-token` and account metadata in `.profile-meta` (email, org, subscription type — captured during `init` and `--refresh`).

### Claude Desktop

Desktop stores its OAuth token encrypted in `config.json` using Electron safeStorage. The encryption key is in the Keychain as `"Claude Safe Storage"` — it's per-machine (not per-account), which is what makes swapping possible.

---

## Token Refresh / Re-Login

When a token expires or gets revoked:

```bash
# Re-login the profile you're currently on
claude-switch --refresh

# Re-login the OTHER profile without switching away
claude-switch --refresh work      # while on personal

# If you already re-logged in manually, just persist the token
claude-switch --save
```

The `--refresh` command runs `claude auth login`, validates the new token, persists it, and updates the profile metadata. When refreshing a different profile, it saves/restores your current keychain token automatically (with a trap for safety).

---

## Directory Layout

```
~/
├── .claude -> .claude-work/              # Symlink to active Claude Code profile
├── .claude-work/                         # Claude Code: work profile
│   ├── .auth-token                       # OAuth token (600 perms)
│   ├── .profile-meta                     # Email, org, subscription type
│   ├── CLAUDE.md                         # Instructions
│   ├── settings.json                     # Config (MCP servers, permissions, env)
│   ├── projects/                         # Project memory
│   ├── history.jsonl                     # Conversation history
│   └── ...
├── .claude-personal/                     # Claude Code: personal profile
│   ├── .auth-token                       # OAuth token (600 perms)
│   ├── .profile-meta                     # Email, org, subscription type
│   └── ...
├── .claude-desktop-profiles/
│   ├── .active                           # Tracks current Desktop profile
│   ├── work/                             # Desktop: work profile snapshot (~5MB)
│   │   ├── config.json                   # Encrypted OAuth token
│   │   ├── Local Storage/                # Account metadata
│   │   ├── Cookies                       # Session cookies
│   │   └── ...
│   └── personal/                         # Desktop: personal profile snapshot
│       └── ...
└── Library/Application Support/Claude/   # Claude Desktop active data (files swapped in place)
```

---

## Gotchas

**`claude auth logout` revokes tokens server-side.** Never use it to switch accounts. Use `claude auth login` directly — it overwrites the keychain without revoking the previous token.

**Running sessions must restart.** Claude Code loads config at startup and caches it in memory. After switching, exit and restart all Claude Code sessions.

**Claude Desktop must be fully quit.** It holds file locks on LevelDB and SQLite files. The script handles quitting automatically.

**Don't delete the "Claude Safe Storage" keychain entry.** This Electron encryption key is shared by both Desktop profiles. Deleting it invalidates both.

**MCP server port conflicts.** If both profiles configure MCP servers on the same ports, kill old servers before switching.

**Project memory is profile-specific.** Each profile has its own `projects/` directory. If you work on the same repo from both profiles, they'll have independent project-level memory.

---

## Development

If you're working on the scripts locally and want to publish changes:

```bash
# Edit scripts in ~/bin (or wherever they live on your PATH)
# Then sync to this repo:
./publish.sh
```

`publish.sh` copies the scripts from `~/bin`, checks for PII leaks (patterns in `.pii-patterns`), and optionally commits and pushes. Create a `.pii-patterns` file with one grep pattern per line (e.g., your email address) — these are checked against the scripts to prevent accidental publication of personal data.

---

## License

MIT
