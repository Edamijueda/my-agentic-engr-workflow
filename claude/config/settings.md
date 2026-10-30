---
verified_against: 2.1.308
verified_on: 2026-10-30
sources:
  - https://code.claude.com/docs/en/settings
  - https://code.claude.com/docs/en/settings-reference
---

# Settings: `settings.json`, scopes, precedence

How Claude Code is configured — the central file is `settings.json`, but there are several scopes, several delivery channels, and a few neighboring files. Distilled from `https://code.claude.com/docs/en/settings`.

## Mental model: scopes + precedence

**Five levels, highest precedence first.** (Four of them are files; command line is a level but not a file.)

| # | Scope | Where | Notes |
| - | --- | --- | --- |
| 1 | **Managed** | Server-managed, MDM/plist, or system `managed-settings.json` | Org policy. Not overridable from below — **except for a short list of keys where a stricter value wins from any scope**; see below. A managed `model` is a starting default, not a lock — the locks are `availableModels` and `deniedModels`. |
| 2 | **CLI arguments** | `--settings <file-or-json>` | Per-session override, inline JSON or a file path. Can set anything user settings can — **but not Managed or Global config keys**. Other flags like `--model` aren't part of this stack; they override their own key for the session. |
| 3 | **Local project** | `.claude/settings.local.json` | Personal, per-repo. Gitignored when Claude Code creates it; **add it yourself if you create it manually**. |
| 4 | **Project shared** | `.claude/settings.json` | Committed to git, team-shared. |
| 5 | **User** | `~/.claude/settings.json` | Personal, all projects. |

**Environment variables aren't a level in this stack.** Where a behaviour has both a shell variable and a settings key, which wins is decided per pair: `ANTHROPIC_MODEL` beats the `model` key from any file, while `ANTHROPIC_DEFAULT_MODEL` applies only when no file sets `model`. An `env` block *inside* a settings file is an ordinary key and follows the levels above.

**Scalar values** (booleans, strings, numbers): higher scope wins.
**Array values** (`permissions.allow`, `sandbox.filesystem.allowWrite`, etc.): **merged and deduplicated across scopes**, not replaced. So a managed `allowWrite: ["/opt/tools"]` plus user `allowWrite: ["~/.kube"]` yields both.

**Four exceptions** where a list doesn't merge:

- `fallbackModel` — position carries meaning, so the highest-precedence file supplies the whole chain.
- `modelPicker` — one ordered list plus a replace flag; never merged. Taken from the highest of managed / `--settings` / user, and **ignored entirely in project and local settings** (v2.1.242+).
- `availableModels` — when managed settings define it, that list applies as-is and user/project/local entries are ignored. Across non-managed scopes it merges normally.
- `modelSettings` — resolved one model at a time, together with `effortLevel`.

## Other config files in the neighborhood

`settings.json` isn't the only file:

| File | Holds |
| --- | --- |
| `~/.claude.json` | OAuth session, MCP servers (user + local scope), per-project state (trust, allowed tools), caches. Setting `settings.json`-typed keys here causes schema errors. |
| `.mcp.json` | Project-scoped MCP server config. |
| `~/.claude/agents/` and `.claude/agents/` | Subagent markdown files. |
| `~/.claude/CLAUDE.md`, `./CLAUDE.md`, `./CLAUDE.local.md` | Memory. See `memory.md`. |
| `.claude/settings.local.json` | Local-scope settings. **In a git repo, read and written at the repository root** even when you start in a subdirectory (a worktree uses the main checkout's root). It stays beside `.claude/settings.json` instead when you're outside a repo, when the repo root is your home directory, or when ownership doesn't match your user. Before v2.1.211 it lived in the starting directory. |

Backups: before each write to `~/.claude.json`, Claude Code saves `~/.claude/backups/.claude.json.backup.<timestamp>` and keeps the five most recent.

## Managed settings (org-deployed)

Multiple delivery channels, all loading the **same JSON shape** (`managed-settings.json`):

1. **Server-managed** — from claude.ai admin console.
2. **MDM / OS policy:**
   - macOS: `com.anthropic.claudecode` preferences domain (plist), deployed via Jamf, Kandji, etc. Other platforms use their own policy mechanism.
3. **File-based:**
   - macOS: `/Library/Application Support/ClaudeCode/managed-settings.json` (Linux: `/etc/claude-code/`)
   - Also supports a `managed-settings.d/` drop-in directory (systemd convention: base file first, drop-ins merged in alphabetical order; arrays concatenate-dedupe, objects deep-merge). Use numeric prefixes (`10-telemetry.json`, `20-security.json`) to control order.

**Within the managed tier**, precedence is: server-managed > MDM/OS > file-based (drop-in + base). Only one managed source loads — they do not merge across tiers.

### Where a lower scope beats managed

For a handful of keys that *restrict* a session, Claude Code honors the stricter value from a scope that otherwise couldn't override managed. The ones likely to matter outside an org:

| Key | Stricter value honored from |
| --- | --- |
| `enableArtifact: false` (or `disableArtifact: true`) | Any scope — and nothing turns artifacts back on (v2.1.242+) |
| `permissions.blockReadsOutsideWorkingDirectories: true` | Any scope (v2.1.257+) |
| `maxEffortLevel` | Any scope, including `--settings` — the **lowest** cap applies (v2.1.267+) |
| `disableClaudeAiConnectors: true`, `isolatePeerMachines: true` | Any scope |
| `useAutoModeDuringPlan: false`, `syncClaudeAiSkills: false`, `syncClaudeAiPlugins: false` | Managed, `--settings`, user, or **local** — but a `false` in shared project settings is ignored |

Full list, including `crossSessionInbound` and `remoteControlAtStartup`: [settings precedence exceptions](https://code.claude.com/docs/en/settings#exceptions-to-managed-settings-precedence).

**Managed parsing is tolerant**: an invalid entry is stripped + logged, the rest enforces.

For user/project/local files the failure mode depends on the damage:

| What's wrong | What happens |
| --- | --- |
| Invalid JSON, or a value the schema rejects | **Settings Error** — a dialog at the start of an interactive session offering to fix it with Claude's help, exit, or continue without that file's settings |
| Individual entries fail (malformed permission rule, unknown hook event) | **Settings Warning** — those values are skipped, the rest of the file stays in effect |
| `~/.claude.json` can't be parsed | **Configuration error** — the file is copied to `~/.claude/backups/.claude.json.corrupted.<timestamp>` and you're asked to fix it by hand or reset |

A `-p` run shows no dialog: it skips the broken file or values and continues, so run `claude doctor` afterwards to see what was dropped.

Settings files are **strict JSON** — a `//` comment or trailing comma is a syntax error.

Security-critical fields (e.g. `allowedMcpServers`, `forceLoginOrgUUID`) fail closed: if the value is wholly invalid, the field acts as the most restrictive interpretation (empty allowlist = block all). Two exceptions: `requiredMinimumVersion` / `requiredMaximumVersion` fail open — a bad version policy can't lock you out.

## `/config` command

`/config` opens a tabbed UI. You can also pass `key=value` to skip the UI:

```
/config verbose=true
/config theme=dark
```

`/status` shows which settings sources are currently loaded. `/doctor` lists settings errors.

## Hot-reload behavior

Most edits apply to the running session — Claude Code watches the files. The `ConfigChange` hook fires for each detected change. Covers `permissions`, `hooks`, `apiKeyHelper`, and most flags.

The hook doesn't fire for managed settings arriving from MDM or the claude.ai console — those reach a running session on a schedule, not on save.

**Read once at session start**, so a mid-session edit doesn't apply:
- `model` — use `/model` instead. Each model has its own prompt cache, so the switch costs one uncached turn.
- `effortLevel` and `modelSettings` — use `/effort` instead.

(`outputStyle` used to belong here. Since v2.1.251 a mid-session switch applies from your next message — see [`../tools/prompt-caching.md`](../tools/prompt-caching.md).)

## Useful keys (curated for solo coding-agent workflow)

The full list is enormous. The keys below are the ones most likely to matter day-to-day. Reach for `/config` and the docs for everything else.

### Authoring + model

| Key | What it does |
| --- | --- |
| `model` | Default model. `--model` and `ANTHROPIC_MODEL` override it for a session, an `availableModels` list still constrains the choice, and an **organization default model wins over your user, project and local files**. |
| `fallbackModel` | Ordered chain tried when the primary is overloaded or unavailable. `"default"` expands. Claude Code keeps at most **three distinct allowed** models and ignores the rest. |
| `effortLevel` | `"low"`, `"medium"`, `"high"`, `"xhigh"`. **Scope changes who it applies to**: in `~/.claude/settings.json` it affects only older models — Opus 5.5 and later ignore it. From project, local, managed, or `--settings` it applies to every model. `/effort` saves per-model under `modelSettings` (v2.1.251+). |
| `maxEffortLevel` | Caps effort (`"max"` = no cap). Lowest cap across scopes wins, including from `--settings`. Per-model caps go inside `modelSettings`. v2.1.267+. |
| `outputStyle` | System-prompt style (e.g. `"Explanatory"`). |
| `alwaysThinkingEnabled` | Thinking is on by default, so `true` does nothing; `false` turns it off. **No effect on models that always think** — Opus 5.5, Sonnet 5.5, Haiku 5.5 and the Fable models — so on a current model this key is inert. `MAX_THINKING_TOKENS` overrides it for one session. |

### Editor + UI

| Key | What it does |
| --- | --- |
| `theme` | `"auto"`, `"dark"`, `"light"`, `"dark-ansi"`, `"dark-daltonized"`, custom themes. |
| `tui` | `"fullscreen"` (alt-screen, flicker-free) or `"default"`. |
| `editorMode` | `"normal"` or `"vim"`. |
| `verbose` | Full tool output instead of truncated summaries. |
| `viewMode` | Default transcript mode at startup. |
| `language` | Force a response language (`"japanese"`, etc.). |
| `axScreenReader` | Screen-reader-friendly flat output. |
| `prefersReducedMotion` | Disable spinners/shimmer/flash. |
| `spinnerTipsEnabled` | Toggle in-spinner tips. |
| `awaySummaryEnabled` | One-line recap when you return. |
| `terminalProgressBarEnabled` | Progress bar in supported terminals (iTerm2 3.6.6+, Ghostty 1.2+, ConEmu). |

### Behavior toggles

| Key | What it does |
| --- | --- |
| `autoCompactEnabled` | Auto-compact when context runs out. Default `true`. Env: `DISABLE_AUTO_COMPACT`. |
| `idleCompaction` | Compact a long conversation left idle on the 1-hour cache TTL. `false` to stop it. v2.1.290+. |
| `promptCacheTtl` / `subagentPromptCacheTtl` | Cache TTL (`5m` or `1h`) for the main conversation and for everything else. **Set `promptCacheTtl: "1h"` on an API key or cloud provider** — you don't get the hour by default there. v2.1.242+. See `../tools/prompt-caching.md`. |
| `switchModelsOnFlag` | `false` to be asked before automatic content-based model fallback instead of switching silently. |
| `autoMemoryEnabled` / `autoMemoryDirectory` | See `memory.md`. |
| `fileCheckpointingEnabled` | Snapshot files before each edit so `/rewind` can restore. |
| `respectGitignore` | Whether `@` file picker respects `.gitignore`. Default `true`. |
| `respondToBashCommands` | After `!` shell commands, set `false` to skip Claude's response (just add output to context). |
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
| `permissions.defaultMode` | Starting permission mode. **Project and local settings can't set `auto` or `bypassPermissions`** — use user or managed settings, or `--permission-mode` for one session. (Before v2.1.257, `bypassPermissions` took effect from any file.) |
| `sandbox.*` | Sandbox config — full section below. |

### Hooks + env + customization

| Key | What it does |
| --- | --- |
| `hooks` | Hook definitions (see hooks doc). |
| `env` | Env vars for every session and spawned subprocess. `NO_COLOR`/`FORCE_COLOR` here apply to subprocesses only; for UI colors set in your shell. |
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

### Plugins (see [`../tools/plugins.md`](../tools/plugins.md))

| Key | What it does |
| --- | --- |
| `enabledPlugins` | `{ "name@marketplace": true/false }`. Project precedence > user; opt out via `.claude/settings.local.json`. |
| `extraKnownMarketplaces` | Register additional plugin marketplaces. |
| `strictKnownMarketplaces` | **Managed only** — allowlist of marketplace sources. |
| `strictPluginOnlyCustomization` | **Managed only** — lock skills/agents/hooks/mcp to plugins + managed. `true` or array of surfaces. |
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

## Cloud sessions read a different subset

A cloud session runs on a fresh clone, not your machine:

- **Not read at all**: `~/.claude/settings.json` and `.claude/settings.local.json` — one stays on your machine, the other isn't in the clone.
- **`.claude/settings.json`**: read in a single-repository session outside a project's threads. In a multi-repository session only `enabledPlugins` and `extraKnownMarketplaces` are read. In a cloud thread inside a project, **none** of it applies.
- **Managed**: a local `managed-settings.json` or MDM profile doesn't reach a cloud session; server-managed settings from the console do.

To change something for a cloud session, set an environment variable on the environment, or commit the key in the single-repository case.

## Global config (`~/.claude.json`) — not in `settings.json`

These keys live in `~/.claude.json` and will error if put in `settings.json`:

| Key | What it does |
| --- | --- |
| `autoConnectIde` | Auto-connect to a running IDE from an external terminal. |
| `autoInstallIdeExtension` | Auto-install Claude Code extension when started from VS Code terminal. Default `true`. |
| `externalEditorContext` | Prepend previous response as `#`-commented context when opening external editor (`Ctrl+G`). |
| `diffTool` | External diff tool. |

Also global-config-only: `claudeInChromeDefaultEnabled`, `copyFullResponse`, `copyOnSelect`, `defaultToAgentsView`, `leftArrowOpensAgents`, `prStatusFooterEnabled`.

**Removed, don't set them:** `teammateDefaultModel` (removed v2.1.234) and `permissionExplainerEnabled` (removed v2.1.257).

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
- Sandbox works on **macOS** (Seatbelt) and **Linux/WSL2** (bubblewrap).
- `failIfUnavailable: true` exits hard if sandbox can't start, instead of silently running unsandboxed.
- Path prefixes: `/` absolute, `~/` home-relative, `./` or no prefix = project root (project settings) or `~/.claude` (user settings). Older `//path` for absolute still works.
- Sandbox merges with permission rules (`Edit`/`Read` allow/deny paths are folded into filesystem rules; `WebFetch(domain:...)` folds into network).
- macOS-only: `allowAppleEvents` (needed for `open`/`osascript`/browser-opens), `enableWeakerNetworkIsolation` (for Go-based tools with MITM proxies).

## Minimum useful `settings.json` for solo use

A starting point — adapt per project. Save as `.claude/settings.json` (commit) or `~/.claude/settings.json` (personal default):

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "claude-sonnet-5-5",
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
