---
verified_against: 2.1.308
verified_on: 2026-10-31
sources:
  - https://code.claude.com/docs/en/sub-agents
---

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
| **`Explore`** | **Opus** on a Claude subscription, Console account, or `ANTHROPIC_BASE_URL` gateway — the model `opus` resolves to. On Bedrock / Google Cloud's Agent Platform / Foundry / Claude Platform on AWS / apps gateway it stays on the main model. | Read-only (no Write/Edit) | File discovery, code search, codebase exploration | Specifies thoroughness on invoke: `quick` / `medium` / `very thorough` |
| **`Plan`** | Inherits | Read-only (no Write/Edit) | Codebase research during plan mode | Used by plan mode for exploration |
| **`general-purpose`** | Inherits | All tools | Complex multi-step research + modification | Default when fork mode enabled but no specific type requested |
| `statusline-setup` | Sonnet | — | When you run `/statusline` | Auto-invoked |
| `claude-code-guide` | Haiku | — | When you ask about Claude Code features | Auto-invoked |
| `claude` | Follows the model resolution order | All subagent tools | Catch-all for work that doesn't fit a specialised agent | — |

> **Explore and Plan skip CLAUDE.md and git status.** For **git status** that's the whole story — you can't change which subagents receive it.
>
> For **CLAUDE.md** there *is* a field: `omitClaudeMd: true` in frontmatter or `--agents` JSON. A subagent with it loads only the managed-policy files — or none at all when the definition itself comes from managed settings. Either way, a subdirectory's `CLAUDE.md` and matching path-scoped rules still load when the subagent touches files there.
>
> The main conversation still has your full CLAUDE.md when it reads the result, so most rules don't need to reach the subagent. For one that must ("ignore `vendor/`"), restate it in the delegation prompt.

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

- **Project subagents**: walked up from CWD — every `.claude/agents/` between CWD and repo root is scanned. Closest-to-CWD wins on name collision.
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
| `omitClaudeMd` | `true` launches the subagent without your user, project and local `CLAUDE.md` — see the note under Built-in subagents. |
| `autoCompactWindow` | Token count from `100000` to `1000000` to make this subagent compact earlier. Write a **plain integer** — `200k` parses as `200`, falls out of range, and is ignored. No effect when the session runs as this agent via `--agent`. `CLAUDE_CODE_AUTO_COMPACT_WINDOW` wins over it. v2.1.296+. |
| `experimental` | Experimental per-agent options, including `cacheTtl`. Read **only from subagent files**, never from `--agents` JSON. |

Multi-word fields are **camelCase**, and an unrecognized field is **ignored without an error** — so a typo does nothing silently. `permissionMode: manual` is an accepted alias for `default`. In `--agents` JSON, `color` and `experimental` are ignored.

> `cd` doesn't persist between Bash/PowerShell calls in a subagent and doesn't affect the parent's working directory. Use `isolation: worktree` for an isolated repo copy.

> Subagents loaded at session start. Direct disk edits need restart. `/agents` interface edits take effect immediately.

> **Combined descriptions have a 15,000-token budget.** Past it Claude Code shows a startup warning and still loads every subagent — nothing is dropped, but the roster is competing for the same context every turn.

> **Subagents auto-compact.** Compaction fires as the subagent's own window fills, and `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` applies to them too. Per-agent tuning is `autoCompactWindow` above.

## Model resolution order

Checked in this order, first match wins (subject to `availableModels` allowlist — excluded values are skipped):

1. Per-invocation `model` parameter when Claude spawns — **aliases only** (`sonnet`/`opus`/`haiku`/`fable`), and a mod's `agent.spawn` hook can override it
2. The definition's `model` frontmatter (`inherit` = main conversation's model)
3. `CLAUDE_CODE_SUBAGENT_MODEL`
4. Main conversation's model

**Before v2.1.251 the env var came first** and overrode both the parameter and the frontmatter. To force one model onto every subagent regardless, set `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=1` alongside it.

## Tools subagents can't use

Subagents inherit the main conversation's tools through **two** filters. **Forks skip both** and get the parent's exact pool.

**Filter 1 — removed from every subagent**, even if named in `tools`:

- `Agent`, once at the depth limit. In a fork the tool stays listed but errors instead of spawning.
- `AskUserQuestion`
- `EndConversation` (it can only end the main conversation)
- `EnterPlanMode`
- `ExitPlanMode` (unless `permissionMode: plan`)
- `ScheduleWakeup`
- `WaitForMcpServers`
- `Workflow`

**Filter 2 — background subagents**, which is **the default** for subagents Claude spawns from the main conversation. They keep only these built-ins: `Read`, `Grep`, `Glob`, `LSP`, `Bash`, `PowerShell`, `Edit`, `Write`, `NotebookEdit`, `WebFetch`, `WebSearch`, `TodoWrite`, `Skill`, `ToolSearch`, `EnterWorktree`, `ExitWorktree`, `Monitor`, `TaskStop`, `SendMessage`, `Artifact`, and `SubagentHandback` where it reports through it. All MCP tools are kept.

> On macOS, Linux and WSL a subagent can **receive `Glob` and `Grep` even when the main conversation doesn't have them** — see [`tools-reference.md`](tools-reference.md).

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

**Restrictions that apply**: `--strict-mcp-config`, `--bare`, managed MCP, `allowedMcpServers`/`deniedMcpServers`. But `--strict-mcp-config` does **not** filter servers passed via `--agents` flag or SDK (explicit caller input).

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

**Permission prompts for background subagents**: surface in the main session, naming which subagent is asking. Esc denies one tool call without stopping the subagent.

When `CLAUDE_CODE_FORK_SUBAGENT=1`, **every** subagent spawn runs in background regardless of the `background` field.

## What loads in a fresh subagent's context

| Component | Loaded? |
| --- | --- |
| **System prompt** | Agent's own prompt + Claude Code's appended env details. NOT the full Claude Code system prompt. |
| **Task message** | The delegation prompt Claude writes when handing off. |
| **CLAUDE.md + memory hierarchy** | Full (user, project, local, managed). **Explore and Plan skip.** |
| **Git status** | A snapshot read **when the subagent starts**, not the parent's start-of-session one. For a subagent in its own worktree, it shows that worktree's branch and commits. Absent in non-git or with `includeGitInstructions: false`. **Explore and Plan skip it, and you can't change that.** |
| **Preloaded skills** | Full content (not just description) of any skill in `skills:`. Built-in agents preload none. |
| **Sibling roster** | A system reminder listing `main` and every other *named* agent — valid `to` values for `SendMessage`. Only appears when the subagent's tools include `SendMessage` and at least one other agent has a name. It's a snapshot at start, so agents named later don't show. |
| **Parent's conversation history** | **NO.** (Exception: forks inherit it.) |
| **Parent's auto memory** | **NO.** |
| **Your output style** | **NO** — a subagent runs its own system prompt. (Exception: forks.) |

Forks are the only subagent type that inherits the parent's conversation history.

> **A custom subagent's prompt replaces the Claude Code system prompt entirely**, the way `--system-prompt` does — unless the agent's prompt is empty. `CLAUDE.md` and project memory still load through the normal message flow even with `omitClaudeMd` set. Running `-p` as a custom agent that has its own prompt also starts **without** the git-status snapshot.

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

## Nested subagents

A subagent can spawn its own subagents. Useful when a delegated task itself splits into parallel subtasks — only the top-level subagent's summary returns to you.

- **Depth limit: 5.** A subagent at depth 5 doesn't receive the Agent tool and can't spawn further. **Limit is not configurable.**
- **Depth fixed at spawn time**. Resuming a subagent later doesn't change its depth.
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

## Forks (enabled by default)

A fork is a subagent that **inherits the entire conversation**. Drops input isolation: same system prompt, tools, model, message history as the main session. Output isolation stays: fork's tool calls don't show in conversation, only its final result returns.

Use when:
- A named subagent would need too much background to be useful.
- You want to try several approaches in parallel from the same starting point.

### Enable / disable

`CLAUDE_CODE_FORK_SUBAGENT=1` enables explicitly (honored in interactive, headless, SDK). `=0` disables everywhere.

When fork mode is enabled:
- Claude can spawn type `fork`. Spawns without a type still use `general-purpose`.
- **Every subagent spawn runs in background**, regardless of the `background` field. Set `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` to keep spawns synchronous.

### Manual fork — `/subtask`, not `/fork`

```
/subtask draft unit tests for the parser changes so far
```

`/subtask` starts a fork whether or not fork mode is on. Named from the first words of the task; appears in the panel below your prompt and runs in the background while you continue.

> **`/fork` does something else.** With agent view on — the default — `/fork` **copies the whole session into a new background session**. It only starts a forked subagent when agent view is turned off.

### Panel controls (running forks)

| Key | Action |
| --- | --- |
| `↑` / `↓` | Move between rows |
| `Enter` | Open transcript + send follow-up |
| `x` | Stop a running fork, or dismiss a row that's no longer running. On the main-session row — or on the row whose transcript you opened with `Enter` — `x` types into the prompt instead |
| `Esc` | Return focus to prompt |

A fork that finishes successfully has its row removed immediately; one that failed or that you stopped keeps its row for 30 seconds.

With a transcript open, follow-up messages and skills go to **that agent**, while built-in commands go to your main conversation:

- `/compact`, `/clear`, `/rewind` act on the main conversation, so you're asked to confirm first.
- `/model` and `/fast` set the main conversation's, not the viewed agent's — they don't run from this view, and a notice says why.
- `Ctrl+Enter` (or `Ctrl+X Ctrl+S`) delivers your message **before** the work the agent is waiting on finishes; anything backgroundable moves to the background and keeps running. v2.1.286+.

### Forks vs named subagents

| | Fork | Named subagent |
| --- | --- | --- |
| Context | Full conversation history | Fresh + task prompt |
| System prompt + tools | Same as main session | From definition |
| Model | Same as main session | From `model` field |
| Permission prompts | Terminal | **Main session** when background |
| Prompt cache | **Shared with main session** | Separate |

**Cache implication**: a fork's first request reuses the parent's prompt cache → cheaper than spawning a fresh subagent for tasks needing the same context. See `prompt-caching.md`.

When Claude spawns a fork via the Agent tool, it can pass `isolation: "worktree"` so file edits go to a separate worktree.

### Fork limitations

- A fork cannot spawn another fork. It can spawn other subagent types (counts toward the depth limit).
- `CLAUDE_CODE_FORK_SUBAGENT=0` disables everywhere including server-side rollout. To block only forks while keeping other subagents, add `Agent(fork)` to `permissions.deny`.
- **While fork mode is on, Claude can't wait for a subagent** in interactive terminal, VS Code or desktop sessions on your machine.
- A fork **shares the parent's prompt cache**, because its system prompt and tool definitions are identical — so for a task needing the same context it costs less than a fresh subagent.
- Claude can pass `isolation: "worktree"` to a fork so its edits land in a separate worktree.

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

## Practical applications for this repo

- **Default to delegating research-heavy tasks**. "Find every place we use the old auth pattern" → Explore subagent. Save the parent context for synthesis.
- **For client projects, build a domain subagent** with `memory: project` so it accumulates project conventions across sessions. Check the memory dir into git (`.claude/agent-memory/`). Personal notes go in `.claude/agent-memory-local/` (gitignored).
- **For the playbook itself** (when we have it): a `playbook-curator` subagent with `tools: Read, Grep, Glob, Edit` and `memory: project` could maintain `shared/principles/` as we add new lessons.
- **`/fork` is the cheapest parallel approach**: same prompt cache as main, full conversation context. Use when you want to test two refactor approaches without redoing setup.

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

Remember `chmod +x`.
