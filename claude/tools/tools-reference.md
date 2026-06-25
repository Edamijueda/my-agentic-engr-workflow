# Tools reference

What Claude can call, with the canonical names used in permission rules, subagent + skill frontmatter, hook matchers, and CLI flags. Distilled from `https://code.claude.com/docs/en/tools-reference`.

## Mental model

**Tool names are the canonical strings.** Wherever you configure tool access, you reference these exact names:

- `permissions.allow` / `permissions.deny` in `settings.json`.
- `/permissions` interactive interface.
- `--allowedTools` / `--disallowedTools` CLI flags.
- Agent SDK `allowedTools` / `disallowedTools`.
- Subagent `tools` / `disallowedTools` frontmatter.
- Skill `allowed-tools` / `disallowed-tools` frontmatter.
- Hook `if` field (uses the parenthesized rule format) and `matcher` field (bare tool name only).

To disable a tool entirely, add its name to `deny`. To add custom tools, connect an MCP server. To add reusable prompts, write a skill — runs via the `Skill` tool, not a new tool entry.

## Catalog

Grouped by purpose. **P** = permission required.

### Files

| Tool | P | Notes |
| --- | --- | --- |
| `Read` | No | Files only (not dirs). Auto-handles images, PDFs, Jupyter notebooks. |
| `Write` | Yes | Whole-file overwrite or new file. No append/merge. |
| `Edit` | Yes | Exact string replacement. Read-before-edit required. |
| `NotebookEdit` | Yes | Jupyter cells by `cell_id`. Uses `Edit(path)` rule format. |
| `Glob` | No | Files by name pattern. **Does NOT** respect `.gitignore` by default. |
| `Grep` | No | File contents (ripgrep regex). **Respects** `.gitignore`. |
| `LSP` | No | Language-server-driven code intelligence. Inactive without a code-intelligence plugin. |

### Shell

| Tool | P | Notes |
| --- | --- | --- |
| `Bash` | Yes | Each command in a fresh process. `cd` carries over within project + `--add-dir` scopes. |
| `PowerShell` | Yes | Native PowerShell. Cross-OS gating below. |
| `Monitor` (v2.1.98+) | Yes | Background watch — feeds each output line to Claude. Uses Bash permission rules. |

### Web

| Tool | P | Notes |
| --- | --- | --- |
| `WebFetch` | Yes | Lossy: fetches a URL, runs your prompt on the content via a fast model, returns the answer. |
| `WebSearch` | Yes | Titles + URLs only — no page fetch. Bare tool name in rules (no specifier). |

### Agent / subagent

| Tool | P | Notes |
| --- | --- | --- |
| `Agent` | No | Spawns a subagent in its own context, or forks the parent conversation. |
| `Skill` | Yes | Executes a skill in the main conversation. |
| `SendMessage` | No | Message an agent-team teammate or resume a subagent by ID. |
| `EnterWorktree` / `ExitWorktree` | No | Git worktree management. |

### Modes / planning

| Tool | P | Notes |
| --- | --- | --- |
| `EnterPlanMode` | No | Switch to plan mode. |
| `ExitPlanMode` | Yes | Present a plan and exit plan mode. |
| `AskUserQuestion` | No | Multi-choice question to user. |

### Task list

| Tool | P | Notes |
| --- | --- | --- |
| `TaskCreate` / `TaskGet` / `TaskList` / `TaskUpdate` / `TaskStop` | No | Session task checklist. |
| `TaskOutput` | No | **Deprecated** — `Read` the task's output file path instead. |
| `TodoWrite` (legacy) | No | **Disabled by default v2.1.142+** in favor of `Task*`. Re-enable with `CLAUDE_CODE_ENABLE_TASKS=0`. |

### Scheduling

| Tool | P | Notes |
| --- | --- | --- |
| `CronCreate` / `CronDelete` / `CronList` | No | Session-scoped scheduled prompts; restored on `--resume` / `--continue` if unexpired. |
| `ScheduleWakeup` | No | Claude calls internally for self-paced `/loop`. Anthropic-only. |
| `RemoteTrigger` | No | Routines on claude.ai. Pro/Max/Team/Enterprise. Anthropic-only. |

### MCP

| Tool | P | Notes |
| --- | --- | --- |
| `ListMcpResourcesTool` | No | List resources from connected MCP servers. |
| `ReadMcpResourceTool` | No | Read a specific MCP resource by URI. |
| `ToolSearch` | No | Loads deferred MCP tools when tool search is enabled. |
| `WaitForMcpServers` (v2.1.142+) | No | Wait for MCP servers still connecting. Only when tool search is disabled. |

### Notifications

| Tool | P | Notes |
| --- | --- | --- |
| `PushNotification` | No | Desktop + phone push when Remote Control is connected. Anthropic-only. |

### Workflows / artifacts / sharing

| Tool | P | Notes |
| --- | --- | --- |
| `Workflow` | Yes | Run a dynamic workflow (orchestrates many subagents). |
| `Artifact` | Yes | Publish HTML/Markdown as a private claude.ai page. Team/Enterprise. |
| `ShareOnboardingGuide` | Yes | Upload `ONBOARDING.md` and return a share link. Pro/Max/Team/Enterprise. |

## Rule formats (per tool)

| Rule format | Applies to |
| --- | --- |
| `Bash(npm run *)` | Bash, **Monitor** |
| `PowerShell(Get-ChildItem *)` | PowerShell |
| `Read(~/secrets/**)` | Read, **Grep**, **Glob**, **LSP** |
| `Edit(/src/**)` | Edit, Write, NotebookEdit |
| `Skill(deploy *)` | Skill |
| `Agent(Explore)` | Agent |
| `WebFetch(domain:example.com)` | WebFetch |
| `WebSearch` | WebSearch (no specifier) |

Two consequences worth remembering:

- **`Edit(path)` also grants read access to the same path** — no matching `Read(...)` rule needed.
- **`Read(path)` deny rules cover Grep, Glob, and LSP too** — they share the path-pattern format.

Tools not listed here (`ExitPlanMode`, `ShareOnboardingGuide`, etc.) accept only the bare tool name. Hook **matcher** fields use bare names — not the parenthesized rule format.

## Per-tool details

### Bash

Each command runs in a fresh process. State persistence:

- **`cd` carries over** when the target is inside the project dir or an `--add-dir` working dir. If `cd` lands outside, Claude Code resets to the project dir and appends `Shell cwd was reset to <dir>` to the result.
- **Disable carry-over**: `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR=1` makes every command start in the project dir.
- **Env vars don't persist** between commands — `export FOO=bar` in one is gone in the next.
- **Aliases + shell functions DO load** — at session start, Claude Code sources `~/.zshrc` / `~/.bashrc` / `~/.profile` (depending on your shell), captures aliases / functions / shell options, applies them to every Bash command.

**For virtualenv/conda**: activate before launching Claude Code. To persist env vars across commands, set `CLAUDE_ENV_FILE` (a shell script) before launch, or populate it from a `SessionStart` hook (see `hooks.md`).

**Limits**:

| Limit | Default | Max | Override |
| --- | --- | --- | --- |
| Timeout | 2 min | 10 min (per-command via `timeout` param) | `BASH_DEFAULT_TIMEOUT_MS`, `BASH_MAX_TIMEOUT_MS` |
| Output length | 30,000 chars | 150,000 chars | `BASH_MAX_OUTPUT_LENGTH` |

When output exceeds the limit, Claude Code saves the full output to a file in the session dir and gives Claude the file path plus a preview from the start. Claude reads or searches the file as needed.

**Background tasks**: `run_in_background: true` for dev servers, watch builds. List/stop with `/tasks`.

### Read

- Always absolute paths.
- Returns content with line numbers.
- Whole-file by default. If file exceeds the token limit, returns the first page with a `PARTIAL view` notice telling Claude how to get more via `offset` / `limit`.
- Explicit `offset` / `limit` that still exceeds → error.

**File types beyond text:**

| Type | Behavior |
| --- | --- |
| Images (PNG, JPG, etc.) | Visual content, not raw bytes. **Resized + recompressed for model image limits** — Claude may see downscaled large screenshots. If pixel detail matters, ask Claude to crop first (ImageMagick via Bash). |
| PDFs | Short → read whole. >10 pages → use `pages: "1-5"` (max 20 pages per call). |
| Jupyter notebooks (`.ipynb`) | All cells with outputs. |

Read only handles files — use `ls` via Bash for directories.

### Edit + Write

**Edit** = exact string replacement. **No regex, no fuzzy matching.** Three checks must all pass:

1. **Read-before-edit**: Claude must have `Read` the file in the current conversation, and the file must not have changed on disk since that read.
2. **Match**: `old_string` must appear in the file exactly as written — one stray whitespace char and the match fails.
3. **Uniqueness**: `old_string` must appear exactly once, OR `replace_all: true`, OR include more surrounding context to pin to one occurrence.

**Read-before-edit via Bash** is accepted for these commands (single file, no pipes, no redirects): `cat`, `head`, `tail`, `sed -n 'X,Yp'`, `grep`, `egrep`, `fgrep`.

> **The "satisfies read-before-edit" list and the "applies to Read/Edit deny rules" list don't fully overlap.** `egrep` and `fgrep` count for read-before-edit but are **NOT** checked against `Read(...)` deny rules. For OS-level enforcement that covers every process, use the sandbox.

**Write** creates new files or overwrites whole files. No append, no merge. **Same read-before-edit constraint applies when overwriting existing files** — new files don't need a prior read. Bash view satisfies it under the same rules as Edit.

For partial changes to existing files, use Edit instead of Write.

### Grep

Built on **ripgrep**, uses ripgrep regex (**not POSIX**). Patterns with regex metacharacters need escaping — `interface{}` in Go matches as `interface\{\}`.

| Output mode | Returns |
| --- | --- |
| `files_with_matches` (default) | File paths only |
| `content` | Matching lines with file + line number |
| `count` | Match count per file |

Scope:
- `glob` parameter: `**/*.tsx`
- `type` parameter: `py`, `rust`, etc.
- `multiline: true` for cross-line patterns

**Respects `.gitignore`** by default. To search a gitignored file, pass its path directly.

### Glob

Standard glob syntax including `**` for recursive matching:

- `**/*.js` — all `.js` at any depth.
- `src/**/*.ts` — all `.ts` under `src/`.
- `*.{json,yaml}` — `.json` and `.yaml` in current dir.

- Sorted by modification time, **capped at 100 files**. Truncation flag if hit.
- **Does NOT respect `.gitignore`** by default — finds gitignored files too. To make Glob respect `.gitignore`: `CLAUDE_CODE_GLOB_NO_IGNORE=false` before launch.

### LSP

Code intelligence via language servers. After each file edit, auto-reports type errors and warnings so Claude can fix without a separate build step.

Direct calls: jump-to-definition, find references, type info, list symbols, search workspace, find implementations, call hierarchies.

**Inactive until you install a code-intelligence plugin** for your language. Plugin bundles config; language server binary installed separately.

### Monitor (v2.1.98+)

Watch something in the background, react to changes mid-conversation. Use cases: tail log + flag errors, poll a PR or CI job, watch a directory, track a long-running script.

Claude writes a small script, runs it in the background, receives each output line as it arrives. Stop by asking Claude to cancel or ending the session.

**Permission rules use `Bash(...)` format** — patterns set for Bash apply.

> **Not available** on Bedrock / Vertex / Foundry. Also not available when `DISABLE_TELEMETRY` or `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` is set.

Plugins can declare monitors that start automatically when the plugin is active.

### NotebookEdit

Targets cells by `cell_id`.

| Mode | Effect |
| --- | --- |
| `replace` (default) | Overwrite cell source |
| `insert` | Add new cell after target. No `cell_id` → insert at start. Requires `cell_type: code` or `markdown`. |
| `delete` | Remove target cell |

Permission rules use `Edit(path)` format. `Edit(notebooks/**)` covers NotebookEdit on those files.

### PowerShell

Native PowerShell. Availability by platform:

| Platform | Default state |
| --- | --- |
| Windows without Git Bash | Auto-enabled |
| Windows with Git Bash | Rolling out progressively |
| Linux / macOS / WSL | Opt-in (requires PowerShell 7+) |

**Enable**:
```json
{ "env": { "CLAUDE_CODE_USE_POWERSHELL_TOOL": "1" } }
```

On Windows, set `=0` to opt out of the rollout. On Linux/macOS/WSL, install `pwsh` and ensure it's on `PATH`.

**Detection**: pwsh.exe (7+) → powershell.exe (5.1) fallback. When enabled, Claude treats PowerShell as the primary shell. Bash tool stays available for POSIX scripts if Git Bash is installed.

**Execution policy**: spawns with `-ExecutionPolicy Bypass` at process scope only — works on default Windows installs without changing machine policy. Does NOT override Group Policy `MachinePolicy` / `UserPolicy` — enterprise lockdowns still apply. To respect the machine's effective policy: `CLAUDE_CODE_POWERSHELL_RESPECT_EXECUTION_POLICY=1`.

**Three places to select PowerShell** (covered fully in `../config/settings.md` and `hooks.md`):

| Where | Setting | Requires tool enabled? |
| --- | --- | --- |
| `settings.json` for interactive `!` commands | `defaultShell: "powershell"` | Yes |
| Individual command hooks | `"shell": "powershell"` | **No** (hooks spawn PS directly) |
| Skill frontmatter for `!` injection | `shell: powershell` | Yes |

Same Bash-style cwd reset behavior applies, including `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR`.

**Preview limitations**: PowerShell profiles not loaded; no sandboxing on Windows.

### WebFetch

Takes URL + prompt. Fetches the page, converts HTML to Markdown, runs the prompt against the content using a small fast model. **Claude receives the model's answer, not the raw page.**

> **Lossy by design.** A result saying the page "doesn't mention X" may only mean the prompt didn't ask about X. To get more detail, ask Claude to fetch again with a more specific prompt, or use `curl` via Bash for the unprocessed page.

Behaviors:
- HTTP → HTTPS upgrade.
- Pages truncated to a fixed char limit before processing.
- **15-minute cache** for repeated URL fetches.
- Redirects to different hosts: returns a text result naming both URLs **without following** — Claude follows up with a second `WebFetch` to the new URL.

**Permission flow**:
- `default` + `acceptEdits` modes: prompts first time per new domain, except for a built-in preapproved doc-domain set.
- `auto` + `bypassPermissions`: skips the prompt entirely.
- Explicit `WebFetch(domain:...)` rule in deny / ask / allow overrides the preapproved set.

Sets `User-Agent: Claude-User...` and prefers Markdown in `Accept` header.

> **Sandbox network rules are separate** — allowing a domain via `WebFetch(...)` doesn't grant sandbox network access. That needs its own `sandbox.network.allowedDomains` entry. See `../config/permissions.md`.

### WebSearch

Anthropic's web search backend. Returns titles + URLs only — **no page fetch**. Follow up with `WebFetch` to read pages.

- Up to 8 backend searches per call (internal refinement).
- `allowed_domains` or `blocked_domains` — **can't combine in a single call**.
- Backend not configurable. For different providers, add an MCP search server.
- Permission rules: bare `WebSearch` only. No specifier.

**Provider availability**:
| Provider | Available? |
| --- | --- |
| Claude API | Yes |
| Microsoft Foundry | Yes |
| Google Vertex AI | Only with Claude 4 models (Opus/Sonnet/Haiku) |
| Amazon Bedrock | **No** — Bedrock doesn't expose the server-side web search tool |

### Agent

Spawns a subagent in a separate context. Subagent returns one text result — **parent does NOT see intermediate tool calls or outputs.**

Same tool also launches **forks** (when fork mode is enabled): forks inherit the parent conversation, always run in background, surface permission prompts in your terminal.

**Tool inheritance** for named subagents (`tools` / `disallowedTools` in subagent frontmatter):

| Set | Subagent gets |
| --- | --- |
| Neither | All parent tools |
| `tools` only | Listed tools only |
| `disallowedTools` only | All parent tools except listed |
| Both | `disallowedTools` wins (removed if in both) |

**Permission prompt behavior**:

- **Foreground subagents**: same prompts you'd see in the main conversation, at the moment of the call.
- **Background subagents (v2.1.186+)**: prompt in main session, naming which subagent. Esc denies that one call without stopping the subagent. **Before v2.1.186**: background subagents auto-denied anything that would prompt.

To limit subagent reach: narrow `tools`, leave Bash off, or set deny rules in settings.

### Smaller / specialty tools

- **`AskUserQuestion`** — multi-choice with up to 4 questions. Each has `question`, `header` (chip label, max 12 chars), `options` (2-4 items), optional `multiSelect`.
- **`EnterPlanMode` / `ExitPlanMode`** — plan mode toggles.
- **`EnterWorktree`** — creates an isolated worktree OR switches into an existing one (with `path`). From within a worktree session, only the `path` form is available and target must be under `.claude/worktrees/`.
- **`ExitWorktree`** — return to original dir. Not available to subagents that already run in their own dir (e.g. `isolation: worktree`).
- **`Skill`** — execute a skill in the main conversation. Permission via `Skill(name)` or `Skill(name *)` rules.
- **`SendMessage`** — agent-team teammate message or subagent resume by ID. Stopped subagents auto-resume in background.
- **`Cron*`** — session-scoped scheduling. Restored on `--resume`/`--continue` if unexpired.
- **`ScheduleWakeup`** — Claude calls internally for self-paced `/loop` (pick next iteration time, 1 min to 1 hour). Pending wakeup appears in `session_crons` in `Stop` hook input. Anthropic-only.
- **`RemoteTrigger`** — backs `/schedule`. Routines on claude.ai. Pro/Max/Team/Enterprise. Anthropic-only.
- **`Workflow`** — dynamic workflow (many subagents, consolidated result).
- **`Artifact`** — publishes HTML/Markdown as private claude.ai page. Team/Enterprise.
- **`PushNotification`** — desktop + (when Remote Control connected) phone push. Anthropic-only.
- **`ShareOnboardingGuide`** — `/team-onboarding` share link. Pro/Max/Team/Enterprise.
- **`ToolSearch` / `WaitForMcpServers`** — MCP infrastructure tools Claude calls automatically.
- **`Task*` family** — session task checklist; covered in their own section above.

### Tools not in this catalog

- **Advisor tool** — server-side (the API runs it), not a Claude Code tool. No name for permission rules or hook matchers.
- **MCP server tools** — added by connecting an MCP server. Naming pattern: `mcp__<server>__<tool>`. Covered in MCP docs (deferred).

## Provider availability quick-table

| Anthropic-only (not Bedrock/Vertex/Foundry) |
| --- |
| `Monitor` |
| `PushNotification` |
| `RemoteTrigger` |
| `ScheduleWakeup` |
| `ShareOnboardingGuide` |
| `Artifact` (also requires Team/Enterprise plan) |

WebSearch availability: Claude API + Foundry yes, Vertex Claude 4 only, **Bedrock no**.

## Environment variables affecting tool behavior

| Var | Tool | Effect |
| --- | --- | --- |
| `BASH_DEFAULT_TIMEOUT_MS` | Bash | Default command timeout (default 120,000). |
| `BASH_MAX_TIMEOUT_MS` | Bash | Hard ceiling on per-command `timeout` parameter. |
| `BASH_MAX_OUTPUT_LENGTH` | Bash | Output cap (default 30,000, max 150,000). |
| `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` | Bash, PowerShell | `=1` disables `cd` carry-over. |
| `CLAUDE_ENV_FILE` | Bash | Shell script to source for persistent env vars. Set before launch. |
| `CLAUDE_CODE_GLOB_NO_IGNORE` | Glob | `=false` makes Glob respect `.gitignore`. |
| `CLAUDE_CODE_USE_POWERSHELL_TOOL` | PowerShell | `=1` enables; on Windows `=0` opts out. |
| `CLAUDE_CODE_POWERSHELL_RESPECT_EXECUTION_POLICY` | PowerShell | `=1` respects effective machine policy instead of `-ExecutionPolicy Bypass`. |
| `DISABLE_TELEMETRY`, `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Monitor | Disables the Monitor tool. |
| `CLAUDE_CODE_ENABLE_TASKS` | TodoWrite / Task* | `=0` re-enables legacy `TodoWrite` (off by default v2.1.142+). |

## Cross-OS notes

- **Bash on Windows** — works via Git Bash. Without Git Bash, PowerShell tool auto-enables and Claude treats PowerShell as the shell.
- **PowerShell on macOS/Linux/WSL** — opt-in via `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` + `pwsh` 7+ on `PATH`.
- **PowerShell preview** — no Windows sandboxing, profiles not loaded.
- **Path rules** — `Read`/`Edit` rules use POSIX-normalized paths even on Windows (`C:\...` → `/c/...`). Covered in `../config/permissions.md`.

## Checking what's loaded

Two ways:

- Ask Claude: `What tools do you have access to?` — conversational summary.
- `/mcp` — exact MCP tool names.

`/permissions` doesn't list tools per se but shows the rules in effect (which is often what you care about).

## How to extend

| You want… | Use |
| --- | --- |
| A custom programmatic tool (Jira, Drive, internal API) | MCP server. Tools appear as `mcp__<server>__<tool>`. |
| A reusable prompt-based workflow | Skill. Runs via the `Skill` tool — no new tool entry. |
| Pre-/post-tool behavior (block, augment, validate) | Hook (see `hooks.md`). |

## Practical applications for this repo

- **`/permissions` denies that matter for any project**: `WebFetch(domain:*)` for high-risk domains; `Bash(curl http*)` and `Bash(wget *)` (since WebFetch alone doesn't stop Bash from hitting URLs); `Read(./.env)` family (covered in `../config/permissions.md`).
- **Watch for `BASH_MAX_OUTPUT_LENGTH` in client projects** with very large logs / generators. Default 30K can truncate and force a file roundtrip; bump to ~80K for log-heavy work, leave default otherwise.
- **`CLAUDE_CODE_GLOB_NO_IGNORE=false`** is worth setting in user-scope env for any monorepo where you genuinely don't want Claude finding gitignored generated files — saves context vs. relying on permissions.
- **For the Windows VM**: PowerShell tool auto-enables without Git Bash. If you install Git Bash later, the tool's rolling out progressively so you may or may not see PowerShell as primary — set `CLAUDE_CODE_USE_POWERSHELL_TOOL=0` to opt out, `=1` to opt in.
- **Cross-platform skill recipes**: prefer `Read` / `Edit` / `Grep` / `Glob` over Bash where possible — those work identically. Bash and PowerShell are the OS-divergent surface.
