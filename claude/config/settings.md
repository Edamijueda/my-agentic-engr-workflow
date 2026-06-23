# Settings: `settings.json`, scopes, precedence

How Claude Code is configured — the central file is `settings.json`, but there are several scopes, several delivery channels, and a few neighboring files. Distilled from `https://code.claude.com/docs/en/settings`.

## Mental model: scopes + precedence

Four scopes, **highest precedence first**:

| # | Scope | Where | Notes |
| - | --- | --- | --- |
| 1 | **Managed** | Server-managed, MDM/registry/plist, or system `managed-settings.json` | Org policy. Can't be overridden by anything below — not even CLI args. |
| 2 | **CLI arguments** | `--settings <file-or-json>` | Per-session override. |
| 3 | **Local project** | `.claude/settings.local.json` | Personal, per-repo. Gitignored when Claude Code creates it; **add it yourself if you create it manually**. |
| 4 | **Project shared** | `.claude/settings.json` | Committed to git, team-shared. |
| 5 | **User** | `~/.claude/settings.json` | Personal, all projects. |

On Windows, `~/.claude` = `%USERPROFILE%\.claude`. Same logical layout, different physical path.

**Scalar values** (booleans, strings, numbers): higher scope wins.
**Array values** (`permissions.allow`, `sandbox.filesystem.allowWrite`, etc.): **merged and deduplicated across scopes**, not replaced. So a managed `allowWrite: ["/opt/tools"]` plus user `allowWrite: ["~/.kube"]` yields both.

Two array exceptions:
- `fallbackModel` — highest-precedence scope supplies the entire chain.
- `availableModels` — when set in managed/policy, lower-precedence entries are replaced (v2.1.175+).

## Other config files in the neighborhood

`settings.json` isn't the only file:

| File | Holds |
| --- | --- |
| `~/.claude.json` | OAuth session, MCP servers (user + local scope), per-project state (trust, allowed tools), caches. Setting `settings.json`-typed keys here causes schema errors. |
| `.mcp.json` | Project-scoped MCP server config. |
| `~/.claude/agents/` and `.claude/agents/` | Subagent markdown files. |
| `~/.claude/CLAUDE.md`, `./CLAUDE.md`, `./CLAUDE.local.md` | Memory. See `memory.md`. |
| `.claude/settings.local.json` | Local-scope settings (also lives under `.claude/`). |

Backups: Claude Code auto-snapshots config files, retains the last 5.

## Managed settings (org-deployed)

Multiple delivery channels, all loading the **same JSON shape** (`managed-settings.json`):

1. **Server-managed** — from claude.ai admin console.
2. **MDM / OS policy:**
   - macOS: `com.anthropic.claudecode` preferences domain (plist), deployed via Jamf, Kandji, etc.
   - Windows: `HKLM\SOFTWARE\Policies\ClaudeCode` registry key (Group Policy / Intune); `HKCU\SOFTWARE\Policies\ClaudeCode` for user-scoped policy (lowest priority in the managed tier).
3. **File-based:**
   - macOS: `/Library/Application Support/ClaudeCode/managed-settings.json`
   - Linux / WSL: `/etc/claude-code/managed-settings.json`
   - Windows: `C:\Program Files\ClaudeCode\managed-settings.json`
   - Also supports a `managed-settings.d/` drop-in directory (systemd convention: base file first, drop-ins merged in alphabetical order; arrays concatenate-dedupe, objects deep-merge). Use numeric prefixes (`10-telemetry.json`, `20-security.json`) to control order.

> **Windows gotcha:** the legacy path `C:\ProgramData\ClaudeCode\managed-settings.json` was dropped in **v2.1.75**. If you ever helped a client deploy there, move to `C:\Program Files\ClaudeCode\`.

**Within the managed tier**, precedence is: server-managed > MDM/OS > file-based (drop-in + base) > HKCU registry. Only one managed source loads — they do not merge across tiers.

**Managed parsing is tolerant**: an invalid entry is stripped + logged, the rest enforces. User/project/local files are strict — a single bad value rejects the whole file.

Security-critical fields (e.g. `allowedMcpServers`, `forceLoginOrgUUID`) fail closed: if the value is wholly invalid, the field acts as the most restrictive interpretation (empty allowlist = block all). Two exceptions: `requiredMinimumVersion` / `requiredMaximumVersion` fail open — a bad version policy can't lock you out.

## `/config` command

`/config` opens a tabbed UI. **As of v2.1.181** you can also pass `key=value` to skip the UI:

```
/config verbose=true
/config theme=dark
```

`/status` shows which settings sources are currently loaded. `/doctor` lists settings errors.

## Hot-reload behavior

Most edits apply to the running session — Claude Code watches the files. The `ConfigChange` hook fires for each detected change. Covers `permissions`, `hooks`, `apiKeyHelper`, and most flags.

**Restart-required keys:**
- `model` — use `/model` mid-session instead.
- `outputStyle` — rebuilds with `/clear` or restart (it's part of the system prompt).

## Useful keys (curated for solo coding-agent workflow)

The full list is enormous. The keys below are the ones most likely to matter day-to-day. Reach for `/config` and the docs for everything else.

### Authoring + model

| Key | What it does |
| --- | --- |
| `model` | Default model (overridden by `--model` and `ANTHROPIC_MODEL`). |
| `fallbackModel` | Ordered chain (max 3) to try when the primary is overloaded. `"default"` expands. |
| `effortLevel` | Persist `/effort` (`"low"`, `"medium"`, `"high"`, `"xhigh"`). |
| `outputStyle` | System-prompt style (e.g. `"Explanatory"`). |
| `alwaysThinkingEnabled` | Default extended thinking on. Disable per-session with `MAX_THINKING_TOKENS=0` in `env`. |

### Editor + UI

| Key | What it does |
| --- | --- |
| `theme` | `"auto"`, `"dark"`, `"light"`, `"dark-ansi"`, `"dark-daltonized"`, custom themes. |
| `tui` | `"fullscreen"` (alt-screen, flicker-free) or `"default"`. |
| `editorMode` | `"normal"` or `"vim"`. |
| `verbose` | Full tool output instead of truncated summaries. |
| `viewMode` | Default transcript mode at startup. |
| `language` | Force a response language (`"japanese"`, etc.). |
| `axScreenReader` | Screen-reader-friendly flat output. v2.1.181+. |
| `prefersReducedMotion` | Disable spinners/shimmer/flash. |
| `spinnerTipsEnabled` | Toggle in-spinner tips. |
| `awaySummaryEnabled` | One-line recap when you return. |
| `terminalProgressBarEnabled` | Progress bar in supported terminals (iTerm2 3.6.6+, Ghostty 1.2+, ConEmu). |

### Behavior toggles

| Key | What it does |
| --- | --- |
| `autoCompactEnabled` | Auto-compact when context runs out. Default `true`. Env: `DISABLE_AUTO_COMPACT`. |
| `autoMemoryEnabled` / `autoMemoryDirectory` | See `memory.md`. |
| `fileCheckpointingEnabled` | Snapshot files before each edit so `/rewind` can restore. |
| `respectGitignore` | Whether `@` file picker respects `.gitignore`. Default `true`. |
| `respondToBashCommands` | After `!` shell commands, set `false` to skip Claude's response (just add output to context). v2.1.186+. |
| `includeGitInstructions` | Strip the built-in commit/PR workflow + git-status from system prompt. `false` if you have your own git skills. |
| `cleanupPeriodDays` | Auto-delete session files older than N days (default 30, min 1). |
| `agent` | Run the main thread as a named subagent (applies its prompt/tools/model). |

### Notifications + remote control

| Key | What it does |
| --- | --- |
| `preferredNotifChannel` | `"auto"`, `"terminal_bell"`, `"iterm2"`, `"iterm2_with_bell"`, `"kitty"`, `"ghostty"`, `"notifications_disabled"`. |
| `remoteControlAtStartup` | Auto-connect Remote Control on every interactive session. |
| `agentPushNotifEnabled` | Push from Remote Control when long tasks finish. |
| `inputNeededNotifEnabled` | Push when a permission prompt is waiting. |

### Permissions + sandbox (covered in detail in `permissions.md`)

| Key | What it does |
| --- | --- |
| `permissions.allow / ask / deny` | Rule arrays. |
| `permissions.additionalDirectories` | Extra working dirs for file access. |
| `permissions.defaultMode` | Default `/permissions` mode. Project/local cannot set `auto` (v2.1.142+). |
| `sandbox.*` | Sandbox config — full section below. |

### Hooks + env + customization

| Key | What it does |
| --- | --- |
| `hooks` | Hook definitions (see hooks doc). |
| `env` | Env vars for every session and spawned subprocess. `NO_COLOR`/`FORCE_COLOR` here apply to subprocesses only (v2.1.143+); for UI colors set in your shell. |
| `allowedHttpHookUrls` | URL patterns HTTP hooks may target. Empty array = block all. |
| `httpHookAllowedEnvVars` | Env var names HTTP hooks may interpolate. |
| `disableAllHooks` | Kill switch. |
| `statusLine` | Custom status line (command-driven). |
| `fileSuggestion` | Custom command for `@` autocomplete (large monorepos). |
| `footerLinksRegexes` | Render clickable badges for ID patterns (e.g. `PROJ-1234` → Jira link). 5 badge max, scheme allowlist, only from user/managed/CLI. |
| `spinnerTipsOverride` / `spinnerVerbs` | Custom spinner text. |

### Memory

| Key | Notes |
| --- | --- |
| `claudeMd` | **Managed only** — content injected as org memory. |
| `claudeMdExcludes` | Glob patterns of CLAUDE.md to skip. Managed-policy files cannot be excluded. |
| `autoMemoryEnabled` / `autoMemoryDirectory` | See `memory.md`. |

### Updates + version pinning

| Key | What it does |
| --- | --- |
| `autoUpdatesChannel` | `"stable"` (≈1 week behind) or `"latest"` (default). Disable entirely with `DISABLE_AUTOUPDATER` in `env`. |
| `minimumVersion` | Soft floor — blocks auto-updates and `claude update` below this. Lets you stay current. |
| `requiredMinimumVersion` / `requiredMaximumVersion` | **Managed only** — hard floor/ceiling, refuses startup outside the range. Recovery: `claude update`/`claude install`/`claude doctor` always work. |

### Plugins (covered in their own doc later)

| Key | What it does |
| --- | --- |
| `enabledPlugins` | `{ "name@marketplace": true/false }`. Project precedence > user; opt out via `.claude/settings.local.json`. |
| `extraKnownMarketplaces` | Register additional plugin marketplaces. |
| `strictKnownMarketplaces` | **Managed only** — allowlist of marketplace sources. |
| `strictPluginOnlyCustomization` | **Managed only** — lock skills/agents/hooks/mcp to plugins + managed. `true` or array of surfaces. v2.1.82+. |
| `disableBundledSkills` | Disable built-in skills + workflows. |
| `disableWorkflows` | Disable dynamic workflows. |

### Worktrees (`--worktree`)

| Key | What it does |
| --- | --- |
| `worktree.baseRef` | `"fresh"` (default, branch from `origin/<default>`) or `"head"` (your local HEAD). |
| `worktree.symlinkDirectories` | Dirs to symlink from main repo into each worktree (e.g. `node_modules`). |
| `worktree.sparsePaths` | Sparse-checkout subset for monorepos. |
| `worktree.bgIsolation` | Background-session isolation: `"worktree"` (default, blocks edits in main) or `"none"`. |

For copying gitignored files like `.env` into worktrees, use a `.worktreeinclude` file, not a setting.

### Attribution

| Key | What it does |
| --- | --- |
| `attribution.commit` | Trailer block for git commits. Empty string hides. |
| `attribution.pr` | PR description footer. Empty string hides. |
| `attribution.sessionUrl` | Append `Claude-Session` trailer / PR link when running from web/Remote Control. Default `true`. |

`includeCoAuthoredBy` is deprecated — use `attribution` instead. To hide everything: set both strings to `""` and `sessionUrl: false`.

## Global config (`~/.claude.json`) — not in `settings.json`

These keys live in `~/.claude.json` and will error if put in `settings.json`:

| Key | What it does |
| --- | --- |
| `autoConnectIde` | Auto-connect to a running IDE from an external terminal. |
| `autoInstallIdeExtension` | Auto-install Claude Code extension when started from VS Code terminal. Default `true`. |
| `externalEditorContext` | Prepend previous response as `#`-commented context when opening external editor (`Ctrl+G`). |
| `teammateDefaultModel` | Default model for agent-team teammates when not specified. |

Pre-v2.1.119 versions also stashed `theme`, `verbose`, `editorMode`, `autoCompactEnabled`, `preferredNotifChannel` here. Modern versions put them in `settings.json`.

## Sandbox settings (summary)

Long subsection in the docs; here are the keys most likely to matter. Full docs: `https://code.claude.com/docs/en/sandboxing`.

```json
{
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "autoAllowBashIfSandboxed": true,
    "excludedCommands": ["docker *"],
    "filesystem": {
      "allowWrite": ["/tmp/build", "~/.kube"],
      "denyRead": ["~/.aws/credentials"]
    },
    "network": {
      "allowedDomains": ["github.com", "*.npmjs.org", "registry.yarnpkg.com"],
      "deniedDomains": ["uploads.github.com"],
      "allowUnixSockets": ["/var/run/docker.sock"],
      "allowLocalBinding": true
    }
  }
}
```

Notes:
- Sandbox works on **macOS, Linux, WSL2**. Native Windows lacks an equivalent — Bash runs unsandboxed.
- `failIfUnavailable: true` exits hard if sandbox can't start, instead of silently running unsandboxed.
- Path prefixes: `/` absolute, `~/` home-relative, `./` or no prefix = project root (project settings) or `~/.claude` (user settings). Older `//path` for absolute still works.
- Sandbox merges with permission rules (`Edit`/`Read` allow/deny paths are folded into filesystem rules; `WebFetch(domain:...)` folds into network).
- macOS-only: `allowAppleEvents` (needed for `open`/`osascript`/browser-opens), `enableWeakerNetworkIsolation` (for Go-based tools with MITM proxies).

## Cross-OS gotchas

- **Path resolution**: `~/.claude` = `%USERPROFILE%\.claude` on Windows. Same JSON, OS-specific path.
- **Managed-settings paths differ per OS** (table above). The Windows legacy `C:\ProgramData\ClaudeCode\` path is gone as of v2.1.75.
- **`defaultShell`**: set to `"powershell"` (and `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` env var) to route `!` commands through PowerShell on Windows. Defaults to `"bash"`. Pair this with the Git-for-Windows recommendation noted in `index.md`.
- **`wslInheritsWindowsSettings`**: Windows-managed only. Makes WSL read managed settings from the Windows policy chain too. Useful for clients who manage Windows policy but devs work in WSL.
- **Sandbox**: macOS uses Seatbelt, Linux/WSL2 uses bubblewrap (`bwrapPath` for managed override), native Windows lacks an equivalent.

## Minimum useful `settings.json` for solo use

A starting point — adapt per project. Save as `.claude/settings.json` (commit) or `~/.claude/settings.json` (personal default):

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "claude-sonnet-4-6",
  "fallbackModel": ["default"],
  "theme": "dark",
  "tui": "fullscreen",
  "editorMode": "normal",
  "verbose": false,
  "autoCompactEnabled": true,
  "fileCheckpointingEnabled": true,
  "permissions": {
    "defaultMode": "default",
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Bash(curl http*)"
    ]
  },
  "env": {
    "CLAUDE_CODE_NEW_INIT": "1"
  }
}
```

The `$schema` line gives autocomplete + inline validation in any editor that understands JSON schema (VS Code, JetBrains, Cursor). Schema lag is normal — recently-added keys may warn until the published schema updates.

## Practical applications for this repo

- The current `CLAUDE.md` at the root is enough for the playbook itself — no `.claude/settings.json` needed yet. Add one when we start enforcing things (e.g. denying access to `~/.claude/projects/.../memory/` from this repo, since that's outside the repo but reachable via `Read`).
- For client projects: drop a checked-in `.claude/settings.json` with `permissions.deny` for `.env` files and high-blast-radius commands. Use `.claude/settings.local.json` for personal toggles (theme, verbose, vim mode).
- On the Windows VM: add `"defaultShell": "powershell"` to `~/.claude/settings.json` so `!` commands work even when Git for Windows isn't installed yet.
- When we eventually have hooks worth keeping, put them in user-scope `settings.json` so they follow you across projects (only project-specific hooks belong in `.claude/settings.json`).

## When to use which file

| Want to... | Put in |
| --- | --- |
| A personal preference for every project (theme, vim mode) | `~/.claude/settings.json` |
| Team-shared rules for one repo (deny `.env`, default mode, hooks) | `.claude/settings.json` |
| Personal override for one repo (don't sync to teammates) | `.claude/settings.local.json` |
| Org policy (you're an admin) | Managed (server / MDM / file-based) |
| Session-only override | `--settings <file-or-json>` on CLI |
| Most behavior you'd otherwise put in chat | `CLAUDE.md` (see `memory.md`) |
| Hard "must / must not" | Hook or `permissions.deny`, not `CLAUDE.md` |
