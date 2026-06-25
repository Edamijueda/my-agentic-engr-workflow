# Subagents

Specialized AI assistants that run in their own context window with custom system prompts, tool access, and permissions. The single biggest token-protection lever — delegate a verbose side-task and only the summary comes back. Distilled from `https://code.claude.com/docs/en/sub-agents`.

## Mental model

A subagent is:

- A **fresh, isolated context window** (no parent history, no skills already invoked, no files Claude already read).
- Spawned via the **Agent tool** when Claude decides to delegate based on the subagent's `description`, or explicitly via natural language, `@`-mention, or `--agent` flag.
- Returns only its **final text response + small metadata trailer** (tokens, duration) to the parent.

**Compare with:**

| | Use this when |
| --- | --- |
| **Subagent** | Verbose side task; tool restrictions; self-contained work that can return a summary. Single session. |
| **Background agents** (separate doc) | Many *independent* sessions running in parallel, monitored from one place. |
| **Agent teams** (separate doc) | Multiple sessions that *communicate* with each other. |
| **Skill** | Reusable prompt/workflow that runs in the main context (no context isolation). |
| **`/btw`** | Quick question against existing conversation context, no tools, answer discarded. |
| **Main conversation** | Frequent back-and-forth; multi-phase shared context; latency matters. |

## Built-in subagents

Always registered in interactive sessions unless denied via `permissions.deny`.

| Agent | Model | Tools | Purpose | Notes |
| --- | --- | --- | --- | --- |
| **`Explore`** | Haiku | Read-only (no Write/Edit) | File discovery, code search, codebase exploration | Specifies thoroughness on invoke: `quick` / `medium` / `very thorough` |
| **`Plan`** | Inherits | Read-only (no Write/Edit) | Codebase research during plan mode | Used by plan mode for exploration |
| **`general-purpose`** | Inherits | All tools | Complex multi-step research + modification | Default when fork mode enabled but no specific type requested |
| `statusline-setup` | Sonnet | — | When you run `/statusline` | Auto-invoked |
| `claude-code-guide` | Haiku | — | When you ask about Claude Code features | Auto-invoked |

> **Explore and Plan are the ONLY subagents that skip CLAUDE.md and git status.** Every other built-in and custom subagent loads both. There's no frontmatter field to change this — restate any rule that must reach the subagent in the delegation prompt.

**Block subagents:**

```json
{ "permissions": { "deny": ["Agent(Explore)", "Agent(my-custom-agent)"] } }
```

Or `"deny": ["Agent"]` to deny the Agent tool entirely (no delegation). For SDK / headless without built-ins: `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1`.

## Scope ladder

Higher priority wins on name collision:

| # | Location | Scope | How to create |
| --- | --- | --- | --- |
| 1 | Managed settings | Org-wide | Deployed via managed settings |
| 2 | `--agents` CLI flag | Session only | JSON when launching |
| 3 | `.claude/agents/` | Project (commit) | `/agents` UI or manual `.md` |
| 4 | `~/.claude/agents/` | All your projects | `/agents` UI or manual `.md` |
| 5 | Plugin `agents/` | Where plugin is enabled | Plugin installs |

**Discovery rules:**

- **Project subagents**: walked up from CWD — every `.claude/agents/` between CWD and repo root is scanned. v2.1.178+: closest-to-CWD wins on name collision.
- **`--add-dir`** directories ALSO have their `.claude/agents/` scanned (exception to the "additional directories grant file access only" rule).
- **Recursive scan** in user + project scopes: organize into subfolders (`agents/review/`, `agents/research/`). **Identity = the `name` frontmatter field only**, not the path. Keep names unique within a scope — duplicates are silently discarded.
- **Plugin scope**: subfolder IS part of the scoped identifier. `agents/review/security.md` in `my-plugin` registers as `my-plugin:review:security`.

**Plugin security caveat**: plugin subagents **ignore** `hooks`, `mcpServers`, `permissionMode` frontmatter (security restriction). Copy the file out to `.claude/agents/` or `~/.claude/agents/` to use them.

## Frontmatter reference

Only `name` and `description` are required.

| Field | Notes |
| --- | --- |
| `name` | Unique within scope. Lowercase + hyphens. Hooks receive this as `agent_type`. Filename doesn't have to match. |
| `description` | What Claude reads to decide whether to delegate. "Use proactively" type phrasing encourages auto-delegation. |
| `tools` | Allowlist. Comma-separated string or YAML list. Inherits all if omitted. To preload Skills, use `skills:` (not `Skill` here). |
| `disallowedTools` | Denylist. Applied first if both set. Then `tools` resolves against remaining pool. Tool in both = removed. |
| `model` | `sonnet`/`opus`/`haiku`/`fable`, full ID, or `inherit`. Default `inherit`. |
| `permissionMode` | `default`/`acceptEdits`/`auto`/`dontAsk`/`bypassPermissions`/`plan`. Ignored for plugin subagents. |
| `maxTurns` | Cap agentic turns. |
| `skills` | Preload skills — **full content injected into context at startup**. Subagent can still invoke unlisted skills via Skill tool. Can't preload `disable-model-invocation: true` skills. |
| `mcpServers` | List of names (reference existing) or inline definitions. Inline = connect at subagent start, disconnect at finish. Ignored for plugin. |
| `hooks` | Lifecycle hooks scoped to this subagent. `Stop` → `SubagentStop` at runtime. Ignored for plugin. |
| `memory` | `user` / `project` / `local` — enables persistent memory dir. |
| `background` | `true` = always run as background task. Default `false`. |
| `effort` | `low`/`medium`/`high`/`xhigh`/`max`. Inherits from session by default. |
| `isolation: worktree` | Run in a temporary git worktree, branched from default branch (not parent HEAD). Worktree auto-cleaned if no changes. |
| `color` | `red`/`blue`/`green`/`yellow`/`purple`/`orange`/`pink`/`cyan` for UI. |
| `initialPrompt` | Auto-submitted as first user turn when running as main session (via `--agent`). Commands + skills processed. Prepended to user prompt. |

> `cd` doesn't persist between Bash/PowerShell calls in a subagent and doesn't affect the parent's working directory. Use `isolation: worktree` for an isolated repo copy.

> Subagents loaded at session start. Direct disk edits need restart. `/agents` interface edits take effect immediately.

## Model resolution order

Checked in this order, first match wins (subject to `availableModels` allowlist — excluded values are skipped):

1. `CLAUDE_CODE_SUBAGENT_MODEL` env var
2. Per-invocation `model` parameter (when Claude spawns)
3. Subagent definition's `model` frontmatter
4. Main conversation's model

## Tools subagents can't use

Inherited tools minus a few that don't apply in a nested context:

- `AskUserQuestion`
- `EnterPlanMode`
- `ExitPlanMode` (unless `permissionMode: plan`)
- `ScheduleWakeup`
- `WaitForMcpServers`

## Tool allow / deny patterns

Both fields accept tool names AND MCP server patterns:

```yaml
# Allowlist — only Read, Grep, Glob, Bash available
tools: Read, Grep, Glob, Bash

# Denylist — everything inherited minus Write and Edit
disallowedTools: Write, Edit

# Whole MCP server denied — keeps other servers + built-ins
disallowedTools: mcp__github

# All MCP servers denied
disallowedTools: mcp__*

# Allow specific MCP server tools
tools: mcp__puppeteer__*

# Both fields: disallowedTools applied first, then tools against remaining
```

## Spawn restrictions (`Agent(type)` syntax)

When an agent runs as the **main thread** with `claude --agent`, the `tools` field controls which subagent types it can spawn:

```yaml
# Allowlist — only worker and researcher can be spawned
tools: Agent(worker, researcher), Read, Bash

# Any subagent (no restrictions)
tools: Agent, Read, Bash

# Cannot spawn any subagents (Agent omitted)
tools: Read, Bash
```

**To block agents while allowing others**: use `permissions.deny` instead.

> In a **subagent definition** (not main thread), `Agent` in `tools` enables nested spawning but `Agent(types)` parens are ignored — you can't restrict from inside a subagent.

> The Task tool was renamed to Agent in v2.1.63. Existing `Task(...)` references in settings and agent definitions still work as aliases.

## MCP servers scoped to a subagent

Define inline servers that connect when the subagent starts and disconnect when it finishes. String references reuse the parent's connection.

```yaml
mcpServers:
  - playwright:                # inline definition
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
  - github                     # reference existing
```

> **Trick for keeping MCP server tools out of the main conversation context**: define inline in the subagent frontmatter rather than in `.mcp.json`. The subagent gets the tools; the parent never sees the descriptions burning context.

**Restrictions that apply (v2.1.153+)**: `--strict-mcp-config`, `--bare`, managed MCP, `allowedMcpServers`/`deniedMcpServers`. But `--strict-mcp-config` does **not** filter servers passed via `--agents` flag or SDK (explicit caller input).

## Permission modes (parent precedence rules)

| Parent mode | Effect on subagent |
| --- | --- |
| `bypassPermissions` | Takes precedence; subagent's `permissionMode` cannot override. |
| `acceptEdits` | Takes precedence; cannot override. |
| `auto` | Subagent inherits auto mode; its `permissionMode` is ignored. Classifier applies same block + allow rules as parent. |
| Other modes | Subagent's `permissionMode` overrides parent. |

## Preload skills

```yaml
skills:
  - api-conventions
  - error-handling-patterns
```

Full skill content (not just description) is injected into the subagent's startup context. Doesn't restrict access — subagent can still discover and invoke project/user/plugin skills via Skill tool during execution.

**To block all skill use**: omit `Skill` from `tools` or add to `disallowedTools`.

**This is the inverse of `context: fork` in a skill** — both use the same underlying system.
- `skills:` on a subagent → subagent controls system prompt, skill content loaded.
- `context: fork` on a skill → skill content drives the prompt, agent type provided by `agent:`.

## Persistent memory

```yaml
memory: project   # recommended default
```

| Scope | Location | Use |
| --- | --- | --- |
| `user` | `~/.claude/agent-memory/<name>/` | Knowledge applies across all projects |
| `project` | `.claude/agent-memory/<name>/` | Project-specific, shareable via git |
| `local` | `.claude/agent-memory-local/<name>/` | Project-specific but gitignored |

When enabled:
- Subagent system prompt automatically includes instructions for reading/writing memory dir.
- First 200 lines / 25KB of `MEMORY.md` injected with instructions to curate if it exceeds.
- Read, Write, Edit tools auto-enabled so subagent can manage memory.

Tips:
- Ask the subagent to consult memory before starting: "Check your memory for patterns you've seen before."
- Ask it to update memory after: "Save what you learned."
- Bake memory instructions into the subagent's markdown body so it proactively maintains its own knowledge.

## Hooks in frontmatter

Defined in the subagent's `.md` file under `hooks:`. Same shape as `settings.json` hooks.

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly-query.sh"
```

- Fire while this subagent is active. Cleaned up when subagent finishes.
- All hook events supported. Most common for subagents: `PreToolUse`, `PostToolUse`, `Stop` (auto-converted to `SubagentStop` at runtime).
- Also fire when the agent runs as the main session via `--agent`.

**Project-level subagent lifecycle hooks** (in `settings.json`):
- `SubagentStart` — matcher: agent type name.
- `SubagentStop` — matcher: agent type name.

Use cases: setup-db-connection on start, cleanup on stop.

## Invoke a subagent

Three patterns, escalating from suggestion to session-wide:

### 1. Natural language (Claude decides)

```
Use the test-runner subagent to fix failing tests
```

### 2. `@`-mention (guaranteed)

```
@"code-reviewer (agent)" look at the auth changes
```

Type `@` for typeahead. Plugin subagents appear under scoped names (`@agent-my-plugin:code-reviewer`). Currently-running named background subagents also appear with status.

> Your full message still goes to Claude; the `@`-mention controls *which* subagent runs, not the prompt it receives. Claude writes the task prompt.

### 3. Whole session as subagent (`--agent`)

```bash
claude --agent code-reviewer
```

Subagent's system prompt **replaces** Claude Code's default. `CLAUDE.md` still loads. Agent name appears as `@<name>` in startup header. Persists across `--resume`.

For plugin agents, full scoped name disambiguates: `claude --agent my-plugin:security-reviewer`. Subfolder included: `my-plugin:review:security`.

Project default — in `.claude/settings.json`:
```json
{ "agent": "code-reviewer" }
```

CLI flag overrides setting.

## Foreground vs background

- **Foreground**: blocks main conversation until complete. Permission prompts pass through to your terminal.
- **Background**: runs concurrently while you continue working.

Claude decides; you can also:
- Ask Claude to "run this in the background".
- Press **Ctrl+B** to background a running task.
- Disable all background: `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`.

**Permission prompts for background subagents (v2.1.186+)**: surface in the main session, naming which subagent is asking. Esc denies one tool call without stopping the subagent. **Before v2.1.186**: background subagents silently auto-denied any prompt.

When `CLAUDE_CODE_FORK_SUBAGENT=1`, **every** subagent spawn runs in background regardless of the `background` field.

## What loads in a fresh subagent's context

| Component | Loaded? |
| --- | --- |
| **System prompt** | Agent's own prompt + Claude Code's appended env details. NOT the full Claude Code system prompt. |
| **Task message** | The delegation prompt Claude writes when handing off. |
| **CLAUDE.md + memory hierarchy** | Full (user, project, local, managed). **Explore and Plan skip.** |
| **Git status** | Start-of-parent-session snapshot. Absent in non-git or with `includeGitInstructions: false`. **Explore and Plan skip.** |
| **Preloaded skills** | Full content (not just description) of any skill in `skills:`. |
| **Parent's conversation history** | **NO.** (Exception: forks inherit it.) |
| **Parent's auto memory** | **NO.** |

Forks are the only subagent type that inherits the parent's conversation history.

## Common patterns

### Isolate high-volume operations

```
Use a subagent to run the test suite and report only the failing tests with their error messages
```

Verbose output stays in subagent context. You get the summary.

### Parallel research

```
Research the authentication, database, and API modules in parallel using separate subagents
```

Each works independently. Claude synthesizes. Works best when paths don't depend on each other.

> When many subagents return detailed results, the *summaries* can still consume significant parent context. For sustained parallelism, use agent teams.

### Chain subagents

```
Use the code-reviewer subagent to find performance issues, then use the optimizer subagent to fix them
```

Claude passes relevant context between sequential subagents.

## Nested subagents (v2.1.172+)

A subagent can spawn its own subagents. Useful when a delegated task itself splits into parallel subtasks — only the top-level subagent's summary returns to you.

- **Depth limit: 5.** A subagent at depth 5 doesn't receive the Agent tool and can't spawn further. **Limit is not configurable.**
- **Depth fixed at spawn time** (v2.1.187+). Resuming a subagent later doesn't change its depth.
- To prevent a specific subagent from spawning others: omit `Agent` from `tools` or add to `disallowedTools`.
- **A fork cannot spawn another fork**, but can spawn other subagent types (counts toward depth limit).

## Resume + transcripts

Each invocation creates a new instance. To continue an existing subagent's work, ask Claude to resume:

```
Use the code-reviewer subagent to review the authentication module
[completes]
Continue that code review and now analyze the authorization logic
```

Claude uses the `SendMessage` tool with the subagent's agent ID. **Built-in Explore and Plan are one-shot and can't be resumed** — use `general-purpose` or a custom subagent.

A stopped subagent that receives a `SendMessage` auto-resumes in the background.

**Transcripts**:
- Location: `~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl`
- Persist independently of main conversation (main compact doesn't affect).
- Auto-cleanup via `cleanupPeriodDays` (default 30).

**Auto-compaction**: subagents support it with same logic as main. `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` applies. Compaction logged in transcript as `compact_boundary` events with `preTokens`.

## Forks (v2.1.117+, enabled by default v2.1.161+)

A fork is a subagent that **inherits the entire conversation**. Drops input isolation: same system prompt, tools, model, message history as the main session. Output isolation stays: fork's tool calls don't show in conversation, only its final result returns.

Use when:
- A named subagent would need too much background to be useful.
- You want to try several approaches in parallel from the same starting point.

### Enable / disable

`CLAUDE_CODE_FORK_SUBAGENT=1` enables explicitly (honored in interactive, headless, SDK). `=0` disables everywhere.

When fork mode is enabled:
- Claude can spawn type `fork`. Spawns without a type still use `general-purpose`.
- **Every subagent spawn runs in background**, regardless of the `background` field. Set `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` to keep spawns synchronous.

### Manual fork

```
/fork draft unit tests for the parser changes so far
```

Named from the first words of the directive. Appears in the panel below your prompt; runs in background while you continue.

### Panel controls (running forks)

| Key | Action |
| --- | --- |
| `↑` / `↓` | Move between rows |
| `Enter` | Open transcript + send follow-up |
| `x` | Dismiss finished / stop running |
| `Esc` | Return focus to prompt |

### Forks vs named subagents

| | Fork | Named subagent |
| --- | --- | --- |
| Context | Full conversation history | Fresh + task prompt |
| System prompt + tools | Same as main session | From definition |
| Model | Same as main session | From `model` field |
| Permission prompts | Terminal | **Main session (v2.1.186+)** when background |
| Prompt cache | **Shared with main session** | Separate |

**Cache implication**: a fork's first request reuses the parent's prompt cache → cheaper than spawning a fresh subagent for tasks needing the same context. See `prompt-caching.md`.

When Claude spawns a fork via the Agent tool, it can pass `isolation: "worktree"` so file edits go to a separate worktree.

### Fork limitations

- A fork cannot spawn another fork. Can spawn other subagent types (counts toward depth limit).
- `CLAUDE_CODE_FORK_SUBAGENT=0` disables everywhere including server-side rollout.

## Subagent vs main vs skill vs `/btw`

Decision tree:

| Want… | Reach for |
| --- | --- |
| Verbose work; only need the summary back | Subagent |
| Tool restrictions on a side task | Subagent (with `tools` allowlist) |
| Same context as parent + parallel approaches | Fork |
| Reusable prompt that runs in main context | Skill |
| Quick question against existing conversation, no tools, answer discarded | `/btw` |
| Frequent back-and-forth iteration | Main conversation |
| Multi-phase work where phases share heavy context | Main conversation |
| Quick targeted change | Main conversation |
| Latency matters | Main conversation (subagents have startup overhead) |

## Cross-OS notes

- **Hook script shells**: PowerShell hooks need `shell: powershell` (see `hooks.md`).
- **Subagent file locations**: same paths on both OSes (`~/.claude/agents/` resolves to `%USERPROFILE%\.claude\agents\` on Windows).
- **Transcript paths**: `~/.claude/projects/...` on both.
- **`isolation: worktree`** works on both — needs git installed.
- **Memory directory paths**: same — `~/.claude/agent-memory/<name>/` per OS.

## Practical applications for this repo

- **Default to delegating research-heavy tasks**. "Find every place we use the old auth pattern" → Explore subagent. Save the parent context for synthesis.
- **For client projects, build a domain subagent** with `memory: project` so it accumulates project conventions across sessions. Check the memory dir into git (`.claude/agent-memory/`). Personal notes go in `.claude/agent-memory-local/` (gitignored).
- **For the playbook itself** (when we have it): a `playbook-curator` subagent with `tools: Read, Grep, Glob, Edit` and `memory: project` could maintain `shared/principles/` as we add new lessons.
- **`/fork` is the cheapest parallel approach**: same prompt cache as main, full conversation context. Use when you want to test two refactor approaches without redoing setup.
- **For the Windows VM**: subagent files are not synced between machines (they live in `~/.claude/agents/` outside the repo). Check team-wide subagents into `.claude/agents/` in this or any project repo and they follow the codebase.

## Example subagents (paste-ready)

### Code reviewer (read-only)

```markdown
---
name: code-reviewer
description: Expert code review specialist. Proactively reviews code for quality, security, and maintainability. Use immediately after writing or modifying code.
tools: Read, Grep, Glob, Bash
model: inherit
---

You are a senior code reviewer ensuring high standards of code quality and security.

When invoked:
1. Run git diff to see recent changes
2. Focus on modified files
3. Begin review immediately

Review checklist:
- Code is clear and readable
- Functions and variables are well-named
- No duplicated code
- Proper error handling
- No exposed secrets or API keys
- Input validation implemented
- Good test coverage
- Performance considerations addressed

Provide feedback organized by priority:
- Critical issues (must fix)
- Warnings (should fix)
- Suggestions (consider improving)

Include specific examples of how to fix issues.
```

### Debugger (can edit)

```markdown
---
name: debugger
description: Debugging specialist for errors, test failures, and unexpected behavior. Use proactively when encountering any issues.
tools: Read, Edit, Bash, Grep, Glob
---

You are an expert debugger specializing in root cause analysis.

When invoked:
1. Capture error message and stack trace
2. Identify reproduction steps
3. Isolate the failure location
4. Implement minimal fix
5. Verify solution works

For each issue, provide:
- Root cause explanation
- Evidence supporting the diagnosis
- Specific code fix
- Testing approach
- Prevention recommendations

Focus on fixing the underlying issue, not the symptoms.
```

### Read-only DB (with hook enforcement)

```markdown
---
name: db-reader
description: Execute read-only database queries. Use when analyzing data or generating reports.
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly-query.sh"
---

You are a database analyst with read-only access. Execute SELECT queries to answer questions.

If asked to INSERT/UPDATE/DELETE/modify schema, explain that you only have read access.
```

```bash
#!/bin/bash
# ./scripts/validate-readonly-query.sh
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
if echo "$COMMAND" | grep -iE '\b(INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|TRUNCATE|REPLACE|MERGE)\b' > /dev/null; then
  echo "Blocked: Write operations not allowed." >&2
  exit 2
fi
exit 0
```

`chmod +x` on macOS/Linux. On Windows, write PowerShell + add `shell: powershell` to the hook.
