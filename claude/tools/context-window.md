# Context window: what fills it, what survives compaction

Token budget is the single biggest lever you have for using Claude Code efficiently. Distilled from `https://code.claude.com/docs/en/context` (an interactive visualization page — content here is the prose plus the inline event descriptions).

## Mental model

Each session has one **context window** that holds:

- The system prompt (Claude's core instructions; you never see it).
- Everything Claude has read from disk (`CLAUDE.md`, files, rule files, auto memory).
- Tool definitions (MCP tool names, skill descriptions).
- Your messages and Claude's responses.
- Tool call outputs (file reads, grep results, command output).
- Hook output that's explicitly forwarded to Claude.

**Default size: 200K tokens.** Extended context (1M tokens) available on Fable 5, Opus 4.6+, and Sonnet 4.6 — select via the `[1m]` model variant.

> Two diagnostic commands: `/context` shows the live breakdown by category with suggestions. `/memory` shows which CLAUDE.md and auto-memory files loaded at startup.

## What loads before you type anything

In order of injection (representative token counts from the docs' sample session — illustrative, not normative):

| What | ~Tokens | Visible in terminal? |
| --- | --- | --- |
| System prompt | ~4,200 | No |
| Auto memory (`MEMORY.md` first 200 lines / 25KB) | ~680 | No |
| Environment info (CWD, platform, shell, OS, git status block) | ~280 | No |
| MCP **tool names** (schemas deferred by default) | ~120 | No |
| **Skill descriptions** (one-line per discoverable skill) | ~450 | No |
| `~/.claude/CLAUDE.md` (user) | ~320 | No |
| Project `CLAUDE.md` | ~1,800 | No |

A few notes on each:

- **Environment info** ends with git branch/status/recent commits in its own block at the very end of the system prompt.
- **MCP tool schemas are deferred by default** — only names are loaded; full schemas load on demand via tool search. Tune with the env var:
  - `ENABLE_TOOL_SEARCH=auto` — load schemas upfront when they fit within 10% of the window.
  - `ENABLE_TOOL_SEARCH=false` — load everything upfront (most expensive).
- **Skill descriptions are the skill index** — Claude needs them to know what's invocable. Skills with `disable-model-invocation: true` are **not in the index** — they stay completely out of context until you invoke them with `/name`. Use this for side-effect-heavy skills (commit, deploy, send-message).
- **Project `CLAUDE.md`** dominates — target under 200 lines. See `claude/config/memory.md` for splitting via `.claude/rules/` and imports.

If you set an `outputStyle` or pass `--append-system-prompt`, those also enter the system prompt at startup.

## What each action adds during the session

| Event | Token cost | Visible? |
| --- | --- | --- |
| Your prompt | ~50 | Full text |
| File read (per file) | ~1–3K (whole content) | One-liner ("Read X.ts") |
| `.claude/rules/` rule with matching `paths:` | ~300–500 | One-liner ("Loaded X.md") |
| Grep / Glob results | ~500–1K | Brief summary |
| Bash command output | full output | Brief summary or truncated |
| Claude's analysis / response text | ~500–1.5K | Full |
| Edit / Write diff | ~400–800 | Full |
| Hook with `additionalContext` | varies | No |

> **File reads dominate context.** Being specific in prompts ("fix the bug in `auth.ts`") matters because it stops Claude from reading three files when one would do.

### Path-scoped rules load on demand

When Claude reads a file matching a `paths:` glob in `.claude/rules/*.md`, that rule file is auto-loaded. You see "Loaded `.claude/rules/api-conventions.md`" in the terminal, but not the rule content — it goes into context silently. Useful for context efficiency: rules that only matter when touching a specific area don't burn tokens otherwise.

### Hook output and context: the additionalContext gotcha

By default a hook's stdout on exit 0 **does not enter context** — it goes to the debug log. To send hook output to Claude, return JSON with `hookSpecificOutput.additionalContext`. That field enters context with no truncation, so keep it concise.

For `PostToolUse` hooks specifically: exit code 2 surfaces stderr as an error to Claude but **cannot block** the tool (it already ran). For pre-tool blocking, use `PreToolUse`.

### Bang commands (`!`)

Typing `!git status` runs the command in your shell. Both the command and its output enter context as part of your next message. Useful for grounding Claude in command results without it running the command itself — e.g. paste a build error in via `!cat build.log` and ask "what's going on?" without Claude re-running the build.

Setting `respondToBashCommands: false` adds the output to context without prompting Claude for a response (v2.1.186+).

## Subagents — the biggest context-saver

A subagent has its **own separate context window**. The main session pays for spawning + reading the result back; nothing the subagent reads from disk touches your main window.

What the subagent gets at spawn:

| | Inherited from main? |
| --- | --- |
| System prompt (shorter than main's) | No — its own |
| Project `CLAUDE.md` | Yes (re-loaded into subagent's window) |
| MCP tool names + skill descriptions | Yes |
| Auto memory | **No** — main session's auto memory not included |
| Conversation history | **No** |
| Task prompt from main | Yes — what Claude wrote to delegate |

Subagent-specific tool gating: most parent tools available, minus a few that don't apply in a nested context (plan-mode controls, background-task tools, and **by default the Agent tool itself** to prevent recursion).

Built-in **Explore** and **Plan** subagents skip the `CLAUDE.md` load for smaller context.

Custom subagents with `memory:` in their frontmatter load their **own separate** `MEMORY.md`.

When the subagent finishes, only its **final text response** plus a small metadata trailer (token counts, duration) comes back. From the docs' sample: a subagent read 6,100 tokens of files; the main session received a 420-token summary. That's the context savings.

> Default to subagents for any "go research X across the codebase" task. The savings compound across multi-step sessions.

## What survives `/compact`

This is the most important table on the page. When the session compacts (manually or automatically), what happens to your instructions depends on how they were loaded:

| Mechanism | After compaction |
| --- | --- |
| System prompt + output style | Unchanged (not in message history) |
| Project-root `CLAUDE.md` + unscoped `.claude/rules/` | **Re-injected from disk** |
| Auto memory (`MEMORY.md`) | **Re-injected from disk** |
| Rules with `paths:` frontmatter | **Lost** until a matching file is read again |
| Nested `CLAUDE.md` in subdirs | **Lost** until a file in that subdir is read again |
| Invoked skill bodies | Re-injected, **capped 5K per skill, 25K total**; oldest dropped first |
| Skill index (descriptions of discoverable skills) | **Not** re-injected after compact (per the inline timeline notes) |
| Hooks | Run as code, not context — n/a |

Two consequences:

- **Skill body truncation keeps the start of the file.** Put the most important instructions near the top of every `SKILL.md`.
- **If a rule must survive compaction**, drop the `paths:` frontmatter or move it into the project-root `CLAUDE.md`. Path-scoped rules trade compaction-survival for per-task loading efficiency.

## Reducing context use — the playbook

| Lever | When to reach for it |
| --- | --- |
| Be specific in prompts | Always. "Fix the bug in `auth.ts`" beats "fix the auth bug." Stops Claude reading 3 files when 1 would do. |
| Delegate research to a subagent | When the task is "go find / understand X across many files." Sub-agent context is free relative to yours. |
| Set `disable-model-invocation: true` on side-effect skills | Skills with commit/deploy/message side effects. Zero index cost until invoked. |
| Keep `CLAUDE.md` under 200 lines | Loaded every session, in full. Long files burn tokens AND reduce adherence. |
| Use `.claude/rules/` with `paths:` | When advice only applies in part of the codebase. Loads on demand. |
| `/compact focus on <topic>` | Before pivoting to a long new task in an existing session. Compacting with a focus preserves what you choose to keep, not what the auto pass guesses. |
| `/clear` between unrelated tasks | Switching domains entirely. Old conversation crowds out the files you need next. |
| `ENABLE_TOOL_SEARCH=auto` | If MCP schemas eat too much startup budget. Defers full schemas until tools are actually needed. |
| `claudeMdExcludes` | In monorepos, to skip other teams' CLAUDE.md files. |
| Extended context (`[1m]` model variant) | When the work genuinely needs a larger window, not a smaller conversation. Fable 5 / Opus 4.6+ / Sonnet 4.6. |
| Auto-compact | Leave on (default). It's the safety net; the other levers are for *before* you hit it. |

## Practical applications for this repo

- **Skills with side effects in `claude/slash-commands/`** (when we have them): always `disable-model-invocation: true`. Slash-only invocation.
- **Cross-cutting principles** ("be concise", "don't add error handling for impossible scenarios") → `~/.claude/CLAUDE.md` (loads everywhere) or this repo's root `CLAUDE.md`.
- **Section-specific guidance** ("when editing setup scripts, use OS-conditional shebangs") → `.claude/rules/setup.md` with `paths: ["claude/setup/**", "shared/project-setup/**"]`. Saves context on every non-setup session.
- **Once we start drafting subagent definitions** (`.claude/agents/`): the built-in Explore/Plan skipping CLAUDE.md is the model — custom agents should also skip CLAUDE.md unless they genuinely need project context.
- **For the Windows VM**: same context model applies; nothing OS-specific in how context fills or compacts.
