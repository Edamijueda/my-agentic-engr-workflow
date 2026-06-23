# Hooks reference

User-defined shell commands, HTTP endpoints, or LLM prompts that fire automatically at lifecycle points. The enforcement and observability layer above `CLAUDE.md` — encodes "from now on, when X happens, do Y" in a way memory and preferences can't. Distilled from `https://code.claude.com/docs/en/hooks`.

## Mental model

Three things to internalize before anything else:

1. **Hooks don't bypass permission rules.** Deny and ask rules are evaluated regardless of what a hook returns. A deny rule always blocks; an ask rule always prompts. A hook returning `"allow"` cannot override either. (Exception: a command hook exiting with code 2 blocks before rules are evaluated — overrides allow rules.)
2. **Five hook types — pick the right one.** Command (shell), HTTP (POST request), MCP tool (call a connected MCP server), prompt (single-turn LLM), agent (subagent with tools). Command is the default; the others have specific use cases.
3. **Hooks run with your full user permissions.** Anything your shell account can do, a hook can do. Validate inputs, quote variables, use absolute paths.

Three configuration layers:

```
Hook event  →  Matcher group  →  Hook handler(s)
  e.g.            e.g.              e.g.
  PreToolUse      Bash              command type, command + args
                                    if: "Bash(rm *)"
                                    timeout: 30
```

## Hook locations (scope ladder)

| Location | Scope | Shareable |
| --- | --- | --- |
| `~/.claude/settings.json` | All your projects | No — local to your machine |
| `.claude/settings.json` | One project | Yes (committed) |
| `.claude/settings.local.json` | One project | No (gitignored when Claude Code creates it) |
| Managed policy settings | Org-wide | Admin-controlled |
| Plugin `hooks/hooks.json` | When plugin enabled | Bundled with plugin |
| Skill or subagent frontmatter | While component is active | Defined in component file |

Managed admins can use `allowManagedHooksOnly` to block user/project/plugin hooks (covered in `claude/config/settings.md`). Plugin hooks force-enabled via managed `enabledPlugins` are exempt.

## Hook handler types (when to use each)

| Type | Use for | Notes |
| --- | --- | --- |
| `command` | Default. Shell script, fast, runs locally. | JSON via stdin, exit code + stdout for decisions. Supports `async`. |
| `http` | Centralized validation service across machines. | POST request body = same JSON. Response body = same decision JSON. Cannot block via status code alone. |
| `mcp_tool` | Pre-built tool on an already-connected MCP server. | Server must already be connected — hook never triggers OAuth/connect flow. Tool output treated like command stdout. |
| `prompt` | LLM evaluates if you want context-aware judgment without writing logic. | Single-turn Haiku call (default). Returns `{ok, reason}` JSON. Default timeout 30s. Doesn't support every event. |
| `agent` | Verification that needs to read files / run shell commands. | **Experimental.** Spawns subagent with Read/Grep/Glob (50-turn cap). Default timeout 60s. |

## Lifecycle events at a glance

28 events. Grouped here by cadence — the key axis when deciding where a hook belongs.

### Per-session

| Event | Fires when | Notable |
| --- | --- | --- |
| `SessionStart` | New session, resume, after `/clear`, after `/compact`. Matcher: `startup` / `resume` / `clear` / `compact`. | Has `CLAUDE_ENV_FILE`; stdout is added as context; can return `additionalContext`, `sessionTitle`, `initialUserMessage`, `watchPaths`, `reloadSkills`. |
| `Setup` | **Only** with `--init-only`, or `--init` / `--maintenance` in `-p` mode. Matcher: `init` / `maintenance`. | For CI prep. Has `CLAUDE_ENV_FILE`. Does **not** fire on normal startup. |
| `InstructionsLoaded` | A CLAUDE.md or `.claude/rules/*.md` loads — at session start and lazily. Matcher: `session_start` / `nested_traversal` / `path_glob_match` / `include` / `compact`. | Observation only — no decision control. Best tool for debugging "why didn't my rule load?" |
| `ConfigChange` | A settings file changes mid-session. Matcher: `user_settings` / `project_settings` / `local_settings` / `policy_settings` / `skills`. | Can block changes (except `policy_settings`). |
| `SessionEnd` | Session terminates. Matcher: `clear` / `resume` / `logout` / `prompt_input_exit` / `bypass_permissions_disabled` / `other`. | Default timeout **1.5s** (overall budget). Cannot block. Override with `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS=5000`. |

### Per-turn

| Event | Fires when | Notable |
| --- | --- | --- |
| `UserPromptSubmit` | Prompt submitted, before Claude processes. | Default timeout **30s** (not 600). Stdout added as context. Can block via `decision: "block"`. |
| `UserPromptExpansion` | User-typed slash command expands. Matcher: command name. | Use for skill/command gating that `PreToolUse Skill` misses. |
| `Stop` | Claude finishes responding (not on interrupt). | `background_tasks` + `session_crons` arrays (v2.1.145+) for distinguishing "done" from "waiting on background work". `decision: "block"` keeps going. Use `additionalContext` for non-error feedback. |
| `StopFailure` | Turn ends due to API error. Matcher: `rate_limit` / `overloaded` / `authentication_failed` / `billing_error` / etc. | Output ignored. Logging / alerting only. |
| `TeammateIdle` | Agent-team teammate about to go idle. | Exit 2 = teammate continues with feedback. JSON `{continue: false}` stops it. |
| `PreCompact` / `PostCompact` | Before/after compaction. Matcher: `manual` / `auto`. | PreCompact can block. Blocking auto-compact that fired to recover from context-limit error surfaces the underlying API error. |

### Per tool call (the agentic loop)

| Event | Fires when | Notable |
| --- | --- | --- |
| `PreToolUse` | Before each tool call. Matcher: tool name. | Decision via `hookSpecificOutput.permissionDecision`: `allow` / `deny` / `ask` / `defer`. Can rewrite via `updatedInput`. |
| `PermissionRequest` | A permission dialog is about to show. Matcher: tool name. | Decision = `behavior: "allow" | "deny"`. Can pre-approve and persist via `updatedPermissions`. |
| `PermissionDenied` | Auto-mode classifier denied. Matcher: tool name. | Set `hookSpecificOutput.retry: true` to tell the model it may retry. Cannot reverse denial. |
| `PostToolUse` | Tool succeeded. Matcher: tool name. | `updatedToolOutput` rewrites what Claude sees (tool already ran — files/network already touched). |
| `PostToolUseFailure` | Tool failed. Matcher: tool name. | `additionalContext` to add diagnostic info for Claude. |
| `PostToolBatch` | After a batch of parallel tool calls resolve, before next model call. No matcher. | Right place for batch-level context (e.g. "you touched these 3 files together — run pytest"). |
| `SubagentStart` / `SubagentStop` | Subagent spawned / finished. Matcher: agent type. | SubagentStop uses same shape as Stop. |
| `TaskCreated` / `TaskCompleted` | `TaskCreate` / task marked complete. No matcher. | Exit 2 blocks the create/complete with stderr to the model. Useful for "tests must pass before TaskCompleted". |

### Observation / specialty

| Event | Use |
| --- | --- |
| `MessageDisplay` | Display-only — strip markdown, redact secrets from rendered text. Default timeout **10s**. Transcript + what Claude sees keep the original. |
| `Notification` | Side-effect on notifications (desktop alerts, etc.). Matcher: notification type. |
| `CwdChanged` | Reactive env management (think direnv). Has `CLAUDE_ENV_FILE`. Can set `watchPaths` for FileChanged. |
| `FileChanged` | Watched file changed on disk. Matcher = literal filenames split on `|` (regex *not* useful here). Has `CLAUDE_ENV_FILE`. |
| `WorktreeCreate` / `WorktreeRemove` | Replace default git-worktree behavior for non-git VCS (SVN, Perforce, Mercurial). WorktreeCreate **must** print absolute path on stdout. Any non-zero exit aborts creation. |
| `Elicitation` / `ElicitationResult` | MCP server requests user input. Matcher: MCP server name. Can answer programmatically. |

## Matcher syntax

The `matcher` field on a group filters when hooks fire. Behavior depends on characters in the value:

| Matcher value | Evaluated as | Example |
| --- | --- | --- |
| `"*"`, `""`, omitted | Match all | Fires on every occurrence |
| Letters, digits, `_`, `|` only | Exact string, or `|`-list of exacts | `Bash`, `Edit|Write` |
| Any other character | JavaScript regex | `^Notebook`, `mcp__memory__.*` |

**Common mistake**: `mcp__memory` (no `.*`) is exact-string and matches no tool. Use `mcp__memory__.*` to match the whole server. (`mcp__github__get_*` matches just the `get_` tools.)

**Canonical tool names only.** Transcript labels (e.g. "Stop Task") differ from canonical names (`TaskStop`). Permission rules and hook matchers both use the canonical name.

Events that **don't support matchers** and always fire: `UserPromptSubmit`, `PostToolBatch`, `Stop`, `TeammateIdle`, `TaskCreated`, `TaskCompleted`, `WorktreeCreate`, `WorktreeRemove`, `CwdChanged`, `MessageDisplay`. Adding a `matcher` to these is silently ignored.

`FileChanged`'s matcher is special — it serves as **both** the literal watch list (split on `|`) and the filter for which hook groups run. Regex doesn't help here.

### `if` field (fine-grained tool/arg filter)

Layered on top of the matcher for tool events (`PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`, `PermissionDenied`). Uses **permission rule syntax** — same shape as `permissions.allow`. Each `if` holds **exactly one rule** — no `&&` / `||` / list. To combine, write multiple hook handlers.

Bash `if` matching has subtleties worth knowing:

| `if` pattern | Bash command | Hook runs? | Why |
| --- | --- | --- | --- |
| `Bash(git *)` | `FOO=bar git push` | Yes | Leading assignments stripped |
| `Bash(git *)` | `npm test && git push` | Yes | Each subcommand checked |
| `Bash(rm *)` | `echo $(rm -rf /)` | Yes | Commands in `$()` / backticks checked |
| `Bash(rm *)` | `echo $(date)` | No | No subcommand matches |
| `Bash(git push *)` | `echo $(date)` | Yes | Patterns specifying more than command name run on `$()` / backticks / `$VAR` (fail-open) |

Unparseable Bash also fails open. **The `if` filter is best-effort** — use the permission system for hard allow/deny.

## Hook handler fields

### Common to all types

| Field | Required | Notes |
| --- | --- | --- |
| `type` | Yes | `"command"`, `"http"`, `"mcp_tool"`, `"prompt"`, or `"agent"` |
| `if` | No | Permission rule — only on tool events. Without it, every handler in the group runs. |
| `timeout` | No | Seconds. Defaults: 600 for command/http/mcp_tool, 30 for prompt, 60 for agent. `UserPromptSubmit` lowers command/http/mcp_tool to 30. `MessageDisplay` lowers it to 10. |
| `statusMessage` | No | Custom spinner message while the hook runs. |
| `once` | No | Run once per session then remove. **Skill frontmatter only** — ignored elsewhere. |

### Command-specific

| Field | Notes |
| --- | --- |
| `command` | Shell command (shell form) or executable path (exec form). |
| `args` | Argument list. Triggers exec form (no shell). |
| `async` | Run in background. Cannot return decisions. See `async` section below. |
| `asyncRewake` | Implies `async`. Exit 2 wakes Claude even if idle. Hook's stderr (or stdout if stderr empty) shown to Claude as system reminder. |
| `shell` | `"bash"` (default) or `"powershell"`. Hooks spawn PowerShell directly — does **not** require `CLAUDE_CODE_USE_POWERSHELL_TOOL`. Ignored when `args` is set. |

### HTTP-specific

| Field | Notes |
| --- | --- |
| `url` | POST endpoint. |
| `headers` | Additional headers. Values support `$VAR` / `${VAR}` interpolation **only for vars in `allowedEnvVars`**. |
| `allowedEnvVars` | Env var allowlist for header interpolation. Required for interpolation to work. |

HTTP error handling differs from command hooks: non-2xx / connection failure / timeout = non-blocking. To block, return 2xx with a JSON body containing the decision fields.

### MCP tool-specific

| Field | Notes |
| --- | --- |
| `server` | Already-connected MCP server name. |
| `tool` | Tool name on that server. |
| `input` | Args passed to tool. String values support `${path}` substitution from hook input (e.g. `"${tool_input.file_path}"`). |

`SessionStart` and `Setup` fire before MCP servers connect, so `mcp_tool` hooks on those events should expect "not connected" errors on first run.

### Prompt / agent-specific

| Field | Notes |
| --- | --- |
| `prompt` | Prompt text. Use `$ARGUMENTS` placeholder for hook input JSON. `\$` for literal `$`. |
| `model` | Defaults to a fast model. |
| `continueOnBlock` | Prompt hooks only. When `ok: false`, feed reason back to Claude and continue instead of stopping (event-dependent — see below). |

## Exec form vs shell form (command hooks)

When does each apply?

- **Exec form** = `args` is set. No shell. `command` resolved as an executable on `PATH`, spawned directly with `args` as the argument vector. Path placeholders substitute as plain strings — **no shell tokenization**, so paths with spaces and special characters need no quoting. Use whenever you reference a `${CLAUDE_*}` path placeholder.
- **Shell form** = `args` omitted. Goes through `sh -c` (macOS/Linux), Git Bash (Windows), or PowerShell (Windows when Git Bash isn't installed). Use when you need pipes, `&&`, redirects, globs, or env var expansion.

```json
{ "type": "command", "command": "node",
  "args": ["${CLAUDE_PLUGIN_ROOT}/scripts/format.js", "--fix"] }
```

```json
{ "type": "command",
  "command": "node \"${CLAUDE_PLUGIN_ROOT}\"/scripts/format.js --fix" }
```

> **Windows gotcha**: exec form needs a real executable. `.cmd` / `.bat` shims (npm, npx, eslint, etc. under `node_modules/.bin`) **cannot** be spawned without a shell. Either invoke `node` + script directly (`"command": "node", "args": ["${CLAUDE_PLUGIN_ROOT}/node_modules/eslint/bin/eslint.js"]`) or use shell form.

> If `command` in exec form is a bare name with whitespace alongside `args`, Claude Code warns at startup — `node script.js` is not a valid executable name.

## Path placeholders

All hooks substitute these in `command` and `args`, and export them as env vars:

| Placeholder | Resolves to |
| --- | --- |
| `${CLAUDE_PROJECT_DIR}` | Project root. Also set in stdio MCP server and plugin LSP env. |
| `${CLAUDE_PLUGIN_ROOT}` | Plugin install dir. Changes on each plugin update. |
| `${CLAUDE_PLUGIN_DATA}` | Plugin persistent data dir. Survives plugin updates — for dependencies, state. |

Plugin hooks also substitute `${user_config.*}` from the plugin's user config.

## Input contract

### Common input fields

All events get:

```json
{
  "session_id": "abc123",
  "transcript_path": "/Users/.../transcript.jsonl",
  "cwd": "/Users/.../project",
  "permission_mode": "default",
  "effort": { "level": "high" },
  "hook_event_name": "PreToolUse"
}
```

In subagent context or with `--agent`, two more fields appear: `agent_id`, `agent_type`.

Tool events also include `tool_name`, `tool_input`, `tool_use_id`. The `tool_input` schema depends on the tool (full schemas in the page).

> **No `$CLAUDE_MODEL` env var.** Only `SessionStart` hooks can receive a `model` field, and it's not guaranteed. Inherited `$ANTHROPIC_MODEL` doesn't track mid-session `/model` switches.

`$CLAUDE_EFFORT` env var is set for tool-context hooks.

### Stdout / response body for command + HTTP hooks

Command hooks: stdin = JSON, stdout = result, exit code = signal.
HTTP hooks: POST body = JSON, response body = result, status code = transport-level signal.

## Output contract — exit codes

For command hooks (HTTP uses status codes — see "HTTP response handling" later):

| Exit code | Behavior |
| --- | --- |
| `0` | Success. Stdout parsed as JSON for decisions. Most events: stdout goes to debug log. `UserPromptSubmit`, `UserPromptExpansion`, `SessionStart`: stdout is added to context. |
| `2` | **Blocking error.** Stdout/JSON ignored. Stderr goes to Claude as error. Effect varies per event (see table below). |
| Other | Non-blocking error. Transcript shows `<hook name> hook error` + first line of stderr. Execution continues. |

> **Exit code 1 is NOT blocking** for most events. Always use `exit 2` for policy enforcement. The exception is `WorktreeCreate` where any non-zero aborts creation.

### Exit 2 effect per event

| Event | Blocks? | Exit 2 effect |
| --- | --- | --- |
| `PreToolUse` | Yes | Blocks the tool call |
| `PermissionRequest` | Yes | Denies the permission |
| `UserPromptSubmit` | Yes | Blocks + erases prompt |
| `UserPromptExpansion` | Yes | Blocks the expansion |
| `Stop` / `SubagentStop` | Yes | Continues the conversation/agent |
| `TeammateIdle` | Yes | Teammate keeps working |
| `TaskCreated` | Yes | Rolls back creation |
| `TaskCompleted` | Yes | Prevents completion |
| `ConfigChange` | Yes | Blocks change (except `policy_settings`) |
| `PostToolBatch` | Yes | Stops the agentic loop before next model call |
| `PreCompact` | Yes | Blocks compaction |
| `Elicitation` | Yes | Denies |
| `ElicitationResult` | Yes | Becomes decline |
| `WorktreeCreate` | Yes | Any non-zero aborts |
| `PostToolUse` / `PostToolUseFailure` | No | Stderr shown to Claude (tool already ran) |
| `PermissionDenied` | No | Exit code ignored. Use JSON `retry: true` to allow model retry. |
| `StopFailure` | No | Output + exit code both ignored |
| `Notification` / `SubagentStart` / `SessionStart` / `Setup` / `SessionEnd` / `CwdChanged` / `FileChanged` / `PostCompact` / `WorktreeRemove` / `InstructionsLoaded` | No | Stderr to user only |
| `MessageDisplay` | No | Original text shown |

## Output contract — JSON

Instead of exit codes, exit 0 and print JSON to stdout for finer control. **Pick one approach per hook — don't mix.** JSON only processed on exit 0.

> Stdout must contain **only** the JSON object. Shell profiles that print on startup will break parsing.

> Output strings (incl. `additionalContext`, `systemMessage`, plain stdout) **capped at 10,000 characters**. Excess saved to file with preview + path returned instead.

### Universal JSON fields

| Field | Default | Behavior |
| --- | --- | --- |
| `continue` | `true` | `false` stops Claude entirely — takes precedence over event-specific decisions. |
| `stopReason` | — | Message shown to user when `continue: false`. Not shown to Claude. |
| `suppressOutput` | `false` | Hides stdout from transcript. Stays in debug log. |
| `systemMessage` | — | Warning to user. |
| `terminalSequence` | — | Allowlisted escape sequence (OSC 0/1/2/9/99/777, BEL) for desktop notifications, window titles, bell. Use instead of writing to `/dev/tty` (unavailable to hooks). v2.1.141+. |

To stop Claude entirely regardless of event:
```json
{ "continue": false, "stopReason": "Build failed" }
```

### `additionalContext` (the big one)

A string from your hook injected into Claude's context window. Wrapped in a system reminder, inserted where the hook fired. Claude reads it on next model request — does not appear as chat.

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "This file is generated. Edit src/schema.ts and run `bun generate` instead."
  }
}
```

**Where the reminder lands:**
- `SessionStart`, `Setup`, `SubagentStart`: at conversation start.
- `UserPromptSubmit`, `UserPromptExpansion`: alongside the submitted prompt.
- `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`: next to the tool result.
- `Stop`, `SubagentStop`: at end of turn (conversation continues so Claude can act on it).

> **Write as factual statements, not imperative system commands.** "The deployment target is production" beats "ALWAYS deploy to production". Out-of-band system-command phrasing triggers Claude's prompt-injection defenses and gets surfaced to you instead of treated as context.

> **For never-changing instructions, prefer `CLAUDE.md`.** Loads without running a script.

> **Resume gotcha**: `additionalContext` is saved to transcript. Resuming with `--continue`/`--resume` **replays the saved text** rather than re-running the hook. Values like timestamps and commit SHAs go stale. `SessionStart` hooks re-run on resume with `source: "resume"` — fine for refreshing.

### Decision control patterns

| Events | Decision shape | Key fields |
| --- | --- | --- |
| `UserPromptSubmit`, `UserPromptExpansion`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`, `Stop`, `SubagentStop`, `ConfigChange`, `PreCompact` | Top-level `decision` | `decision: "block"`, `reason`. Stop/SubagentStop also accept `hookSpecificOutput.additionalContext` for non-error feedback. |
| `PreToolUse` | `hookSpecificOutput` | `permissionDecision: "allow"|"deny"|"ask"|"defer"`, `permissionDecisionReason`, optional `updatedInput`. |
| `PermissionRequest` | `hookSpecificOutput.decision` | `behavior: "allow"|"deny"`, optional `updatedInput`, `updatedPermissions`. |
| `PermissionDenied` | `hookSpecificOutput` | `retry: true` to tell model it may retry. |
| `MessageDisplay` | `hookSpecificOutput` | `displayContent` replaces rendered text (display-only). |
| `WorktreeCreate` | Path return | Command stdout = path. HTTP = `hookSpecificOutput.worktreePath`. |
| `Elicitation`, `ElicitationResult` | `hookSpecificOutput` | `action: "accept"|"decline"|"cancel"`, `content` (form values). |
| `SessionStart`, `Setup`, `SubagentStart` | Context only | `additionalContext`. SessionStart also: `initialUserMessage`, `watchPaths`, `sessionTitle`, `reloadSkills`. |
| `WorktreeRemove`, `Notification`, `SessionEnd`, `PostCompact`, `InstructionsLoaded`, `StopFailure`, `CwdChanged`, `FileChanged` | None | Side effects only. |

### `PreToolUse` decision control in detail

The most flexible event. Four outcomes:

| `permissionDecision` | Effect |
| --- | --- |
| `"allow"` | Skip permission prompt. Deny/ask rules **still apply**. |
| `"deny"` | Block the call. |
| `"ask"` | Show user a permission prompt. Dialog includes `[User]` / `[Project]` / `[Plugin]` / `[Local]` label so user knows where ask came from. |
| `"defer"` | **For SDK/headless only.** Exits session gracefully with `stop_reason: "tool_deferred"` so the calling process can collect input through its own UI, then resume. Logged-and-ignored in interactive mode. v2.1.89+. |

Multiple hook precedence: `deny` > `defer` > `ask` > `allow`.

**Tool input rewriting**: `updatedInput` replaces the entire input object — include unchanged fields too. Combine with `"allow"` to auto-approve modified input, or `"ask"` to show the user the modified input.

**For `AskUserQuestion` and `ExitPlanMode` in headless mode**: returning `"allow"` alone hangs. Use `updatedInput` to supply the answer too (for `AskUserQuestion`, echo back `questions` and add `answers` mapping each question text to chosen label).

### `PostToolUse` `updatedToolOutput`

Rewrites what Claude sees after a tool runs. Must match the tool's output shape (e.g. `Bash` returns `{stdout, stderr, interrupted, isImage}`). For built-in tools, mismatched shape is silently ignored and original is used. MCP tool output passes through without schema validation.

> **Tool already ran.** Files written, commands executed, network requests sent have already happened. Telemetry (OTel spans, analytics) also captures the original output. To prevent or modify a call before it runs, use `PreToolUse`.

> Stripping error details Claude needs can cause it to proceed on a false assumption — be conservative with redaction.

## Per-event specifics worth knowing

### `SessionStart` — the most powerful event

Stdout adds to context directly — no JSON needed. JSON unlocks more:

| Field | Effect |
| --- | --- |
| `additionalContext` | Added to context at conversation start. |
| `initialUserMessage` | Becomes first user message of the session. Applies in `-p` (headless). |
| `sessionTitle` | Sets title (like `/rename`). Only on `source: "startup"` or `"resume"`. |
| `watchPaths` | Absolute paths for `FileChanged` to watch this session. |
| `reloadSkills: true` | Re-scans skill/command dirs after hooks complete — skills the hook installed work in the same session. |

`SessionStart`, `Setup`, `CwdChanged`, and `FileChanged` hooks have access to `CLAUDE_ENV_FILE` env var — write `export VAR=value` lines there and they persist into every subsequent Bash command. (`>>` not `>` so other hooks' vars aren't clobbered.)

### `Stop` — `additionalContext` vs `decision: "block"`

| Approach | Transcript shows | Use for |
| --- | --- | --- |
| `decision: "block"` + `reason` | Hook error / continuation feedback | Hard "you can't stop yet" — e.g. tests must pass. |
| `hookSpecificOutput.additionalContext` | Hook feedback (clean) | "Working as designed" guidance — e.g. "run tests before finishing". |

Both honor the 8-consecutive-continuation cap and the `stop_hook_active` input flag.

`background_tasks` / `session_crons` arrays (v2.1.145+) help distinguish:
- Session is done → empty arrays.
- Session paused waiting for background work → entries describe what's still running.

### `PreCompact` blocking nuance

Blocking a `manual` compact is fine. Blocking an `auto` compact:
- If triggered proactively before the limit → conversation continues uncompacted.
- If triggered to recover from a context-limit API error → the underlying error surfaces and the current request **fails**.

Don't unconditionally block auto-compact.

### `PermissionRequest` — permission update entries

The `updatedPermissions` array (and `permission_suggestions` input field) use entries with these types:

| `type` | Effect |
| --- | --- |
| `addRules` / `replaceRules` / `removeRules` | Add/replace/remove permission rules. |
| `setMode` | Change permission mode (`default`/`auto`/`acceptEdits`/`dontAsk`/`bypassPermissions`/`plan`). `bypassPermissions` only takes effect if session was launched with bypass-enabling flags. |
| `addDirectories` / `removeDirectories` | Add/remove working directories. |

Each entry has a `destination`: `session` (in-memory, discarded on exit), `localSettings`, `projectSettings`, or `userSettings`.

A hook can echo one of the `permission_suggestions` it received as its own `updatedPermissions` — equivalent to the user clicking "always allow" in the dialog.

## Hooks in skills and agents

Hooks can be declared in skill or subagent **frontmatter** — scoped to the component's lifetime, cleaned up when it finishes. For subagents, `Stop` hooks are auto-converted to `SubagentStop`.

```yaml
---
name: secure-operations
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/security-check.sh"
---
```

`once: true` only honored in skill frontmatter — useful for "verify once per session" patterns.

## Async hooks (`async: true`)

Command hooks only. Hook starts and Claude immediately continues without waiting.

**Cannot return decisions** — `decision`, `permissionDecision`, `continue` all ignored since the triggering action already proceeded.

When the background process exits, `additionalContext` is delivered on the **next conversation turn**. If the session is idle, the response waits until the next user interaction — **unless** `asyncRewake: true` and exit code 2, which wakes Claude immediately.

`timeout` (default 600s, max) limits how long the background process runs.

No deduplication — each firing creates a separate process.

Notifications suppressed by default. Enable with `Ctrl+O` or `--verbose`.

## Prompt-based hooks (`type: "prompt"`)

LLM evaluates instead of a script. Returns:

```json
{ "ok": true | false, "reason": "Explanation" }
```

`continueOnBlock: true` lets `PostToolUse` and `TeammateIdle` feed the reason back instead of stopping.

**Supported events** (full set with prompt + agent): `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`, `PermissionRequest`, `PermissionDenied`, `Stop`, `SubagentStop`, `TaskCreated`, `TaskCompleted`, `TeammateIdle`, `UserPromptSubmit`, `UserPromptExpansion`.

**NOT supported**: most observation events (`ConfigChange`, `CwdChanged`, `FileChanged`, `InstructionsLoaded`, `Notification`, etc.), `SessionStart`, `Setup`, `WorktreeCreate`/`Remove`, `Elicitation`/`Result`, `PreCompact`/`PostCompact`, `SessionEnd`, `StopFailure`, `SubagentStart`.

For `PermissionDenied` and `PermissionRequest`, prompt/agent hooks run but their output is discarded. Use command hooks for actual decision control on those.

## Agent-based hooks (`type: "agent"`) — experimental

Subagent with tool access (Read, Grep, Glob, etc.). Multi-turn — 50-turn cap. Same response schema as prompt hooks. Default timeout 60s.

Use when verification needs to inspect actual files or test output, not just the hook input.

> **Experimental.** Behavior and config may change. Prefer command hooks for production.

## Cross-OS notes

### Windows shell

Set `"shell": "powershell"` on the individual hook — hooks spawn PowerShell directly, doesn't require `CLAUDE_CODE_USE_POWERSHELL_TOOL`. Auto-detects `pwsh.exe` (PS 7+) → `powershell.exe` (5.1) fallback.

```json
{ "type": "command", "shell": "powershell",
  "command": "Write-Host 'File written'" }
```

For exec form on Windows: see the **`.cmd` / `.bat` shim gotcha** above. Use `node` + script-path pattern, or shell form.

### macOS / Linux session isolation (v2.1.139+)

Command hooks run in their own session **without a controlling terminal**. Can't open `/dev/tty` or send escape sequences directly. Use:
- `systemMessage` JSON field to surface a message to the user.
- `terminalSequence` JSON field for desktop notifications / bell / window title.

Windows has no `/dev/tty` either — same path applies.

### `/dev/tty` replacement (`terminalSequence`)

Allowlist: OSC `0`, `1`, `2` (window/icon titles), OSC `9` (iTerm2/ConEmu/Windows Terminal/WezTerm notifications + `9;4` taskbar progress), OSC `99` (Kitty), OSC `777` (urxvt/Ghostty/Warp), bare BEL. Anything else (CSI cursor/color, OSC palette, OSC 8 hyperlinks, OSC 52 clipboard, OSC 1337) is rejected.

Build escape strings with `printf` octal escapes so control bytes never appear on the command line, then `jq -n --arg` to build JSON.

## Settings keys related to hooks

| Key | Notes |
| --- | --- |
| `disableAllHooks: true` | Kill switch. Respects managed precedence. |
| `allowManagedHooksOnly` | Managed only. Locks down to managed/SDK/managed-plugin hooks. |
| `allowedHttpHookUrls` | URL pattern allowlist for HTTP hooks. Empty array blocks all HTTP hooks. Arrays merge across scopes. |
| `httpHookAllowedEnvVars` | Env var name allowlist for HTTP header interpolation. Arrays merge. |
| `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` (env) | Override default 1.5s budget for SessionEnd hooks. |
| `CLAUDE_CODE_DEBUG_LOG_LEVEL=verbose` (env) | More granular hook matching log lines. |

## The `/hooks` menu

Type `/hooks` in-session to browse configured hooks. Read-only — shows event, matcher, type, source file (`User` / `Project` / `Local` / `Plugin` / `Session` / `Built-in`), full command / prompt / URL. To modify, edit settings JSON or ask Claude.

## Debug

Hook execution writes to the debug log: matched/exit codes/full stdout/stderr.

```
claude --debug                    # writes to ~/.claude/debug/<session-id>.txt
claude --debug-file /tmp/dbg.log  # custom path
CLAUDE_CODE_DEBUG_LOG_LEVEL=verbose claude --debug   # adds matcher counts
```

`--debug` does **not** print to terminal — read the file.

## Security checklist

- Quote shell variables (`"$VAR"` not `$VAR`).
- Check for `..` in file paths (block path traversal).
- Use absolute paths (`${CLAUDE_PROJECT_DIR}/.claude/hooks/foo.sh`, not `./foo.sh`).
- Skip sensitive files (`.env`, `.git/`, keys).
- Validate JSON input — don't trust blindly.
- Hooks run with **full user permissions**. Anything you can do, they can do.

## Practical recipes

### Block `rm -rf` regardless of mode

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(rm -rf *)",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rmrf.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

```bash
#!/bin/bash
jq -n '{
  hookSpecificOutput: {
    hookEventName: "PreToolUse",
    permissionDecision: "deny",
    permissionDecisionReason: "rm -rf is blocked by hook"
  }
}'
```

### Auto-format after every edit

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "if": "Edit(*.ts)|Write(*.ts)",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/format.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

### Inject git status into every session

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/git-context.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

```bash
#!/bin/bash
BRANCH=$(git branch --show-current 2>/dev/null)
CHANGED=$(git diff --name-only 2>/dev/null | head -10)
jq -nc --arg b "$BRANCH" --arg c "$CHANGED" '{
  hookSpecificOutput: {
    hookEventName: "SessionStart",
    additionalContext: ("Current branch: " + $b + "\nChanged files:\n" + $c)
  }
}'
```

### Run tests after edits — non-blocking

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/run-tests.sh",
            "args": [],
            "async": true,
            "asyncRewake": true,
            "timeout": 300
          }
        ]
      }
    ]
  }
}
```

`asyncRewake` wakes Claude on test failure (exit 2) even if you've stepped away.

### Block `git push` without explicit user confirmation

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(git push *)",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/confirm-push.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

```bash
#!/bin/bash
jq -n '{
  hookSpecificOutput: {
    hookEventName: "PreToolUse",
    permissionDecision: "ask",
    permissionDecisionReason: "Push requires explicit confirmation"
  }
}'
```

### Verify tests pass before TaskCompleted

```json
{
  "hooks": {
    "TaskCompleted": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/tests-pass.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

```bash
#!/bin/bash
if ! npm test >&2; then
  echo "Tests failing — task cannot be marked complete." >&2
  exit 2
fi
exit 0
```

## Practical applications for this repo

- Right now there's nothing to hook into — the playbook is mostly docs. Once we add scripts under `claude/setup/`, `shared/project-setup/`, or example workflows, candidates emerge:
  - `SessionStart` hook in `~/.claude/settings.json` that loads current branch + recent commits as context (universally useful).
  - `PreToolUse` deny for `Bash(rm -rf *)` and `Bash(curl http*)` in user-scope settings.
  - `InstructionsLoaded` hook (debug only) when troubleshooting why a rule isn't loading.
- For the Windows VM: the **`.cmd` / `.bat` shim gotcha** is the most likely paper cut when writing cross-OS hook examples. Note in `claude/setup/` whenever we add Windows install steps.
- Personal cache-aware tweak: any hook returning `additionalContext` in `PostToolUse` will inject content into the conversation, which doesn't itself invalidate the prompt cache (it's appended). But denying a bare tool name in settings mid-session **does** invalidate (see `claude/tools/prompt-caching.md`). Worth noting next to any "from now on" workflow.
