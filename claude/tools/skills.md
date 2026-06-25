# Skills

How to package repeatable workflows so Claude (or you) can invoke them with `/name`. Distilled from `https://code.claude.com/docs/en/skills`.

## Mental model

A skill is a directory with a `SKILL.md` file. The frontmatter tells Claude when the skill is relevant; the markdown content is the instructions Claude follows when it runs. Claude can auto-invoke skills when their description matches, or you can invoke directly with `/skill-name`.

**Custom commands and skills are the same mechanism now.** A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and work the same way. Skills add: a directory for supporting files, frontmatter for invocation control, and automatic discovery by Claude.

When to reach for a skill vs. other mechanisms:

| You want… | Use |
| --- | --- |
| A multi-step procedure invoked on demand (`/commit`, `/deploy`) | Skill (often with `disable-model-invocation: true`) |
| Always-on context for the project (build commands, conventions) | `CLAUDE.md` (see `claude/config/memory.md`) |
| Always-on guidance only when working in a specific area | `.claude/rules/*.md` with `paths:` frontmatter |
| Hard "must / must not" enforcement | Hook or `permissions.deny` (not a skill) |
| Background knowledge Claude should know but never trigger a command | Skill with `user-invocable: false` |

**Why skills beat CLAUDE.md for long reference material**: a skill's body loads only when used, so a 500-line API reference inside a skill costs basically nothing until Claude invokes it. The same content in `CLAUDE.md` loads on every session.

## Where skills live (scope ladder)

| Location | Path | Applies to | Override order |
| --- | --- | --- | --- |
| Enterprise | Managed settings | All users in org | Wins over everything below |
| Personal | `~/.claude/skills/<name>/SKILL.md` | All your projects | Wins over project/bundled |
| Project | `.claude/skills/<name>/SKILL.md` | This project only | Wins over bundled |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | Where plugin enabled | Namespaced (`plugin-name:skill-name`) — never conflicts |
| Bundled | Ships with Claude Code | Default | Lowest precedence |

A `code-review` skill in your project's `.claude/skills/` replaces the bundled `/code-review`. Plugin skills don't conflict with anything (they're namespaced).

**Nested skills**: `.claude/skills/` directories under your CWD are loaded on demand when Claude touches files in those subdirs. In a monorepo with `apps/web/.claude/skills/deploy/`, the nested one appears as `/apps/web:deploy` and Claude picks whichever variant matches the files it's working on.

Files in `.claude/commands/` (the legacy custom-command location) still work and use the same frontmatter — skills add directory + supporting files.

## `SKILL.md` anatomy

Two parts: YAML frontmatter between `---` markers + markdown content.

```yaml
---
description: Summarizes uncommitted changes and flags anything risky.
disable-model-invocation: false
---

## Current changes

!`git diff HEAD`

## Instructions

Summarize the changes above in 2-3 bullet points...
```

The directory name (`summarize-changes`) becomes the command (`/summarize-changes`). The `description` is what Claude reads to decide whether to invoke the skill.

> **Keep `SKILL.md` under 500 lines.** Move large reference material to supporting files in the same dir and link from `SKILL.md`. Once invoked, the body stays in context for the rest of the session — every line is a recurring token cost.

## Frontmatter reference

All fields optional except `description` (strongly recommended — without it, Claude won't know when to use the skill).

| Field | Notes |
| --- | --- |
| `name` | Display label in listings. **Doesn't change the command** (directory name does — except for plugin-root SKILL.md). |
| `description` | What the skill does + when to use it. Truncated at **1,536 characters** combined with `when_to_use`. Put the key use case first. |
| `when_to_use` | Appended to `description` for matching. Same 1,536-char cap. |
| `argument-hint` | Autocomplete hint, e.g. `[issue-number]`. |
| `arguments` | Named positional args for `$name` substitution. Space-separated string or YAML list. |
| `disable-model-invocation` | `true` = only you can invoke (description not in Claude's context). Use for skills with side effects. |
| `user-invocable` | `false` = hidden from `/` menu; only Claude can invoke. Use for background knowledge. |
| `allowed-tools` | Pre-approved tools while skill is active. Space- or comma-separated, or YAML list. Workspace-trust gate for project-level. |
| `disallowed-tools` | Tools removed from Claude's pool while skill is active. Clears when you send next message. |
| `model` | Override model for this turn only. Session model resumes on next prompt. `inherit` keeps active. |
| `effort` | Override effort level for this turn. |
| `context: fork` | Run in a forked subagent. See "Run in a subagent" below. |
| `agent` | Which subagent type to use when `context: fork` is set. |
| `hooks` | Hooks scoped to the skill's lifecycle. Format = same as `settings.json` hooks. |
| `paths` | Glob patterns. Skill only auto-loads when working with matching files. Same format as path-scoped rules. |
| `shell` | `bash` (default) or `powershell` for `!`-injection commands. PowerShell requires `CLAUDE_CODE_USE_POWERSHELL_TOOL=1`. |

## How the command name is chosen

| Skill location | Command name source |
| --- | --- |
| `~/.claude/skills/foo/SKILL.md` or `.claude/skills/foo/SKILL.md` | Directory name → `/foo` |
| Nested + name collision (`apps/web/.claude/skills/deploy/`) | Path-qualified → `/apps/web:deploy` |
| `.claude/commands/foo.md` (legacy) | Filename without ext → `/foo` |
| Plugin `skills/foo/` subdirectory | Namespaced → `/plugin-name:foo` |
| Plugin-root `SKILL.md` | Frontmatter `name`, falls back to plugin dir name → `/plugin-name` |

**Plugin-root is the only case where `name` sets the command** (no skill subdir to take it from).

## String substitutions

In `SKILL.md` content:

| Variable | Expands to |
| --- | --- |
| `$ARGUMENTS` | Full argument string as typed. |
| `$ARGUMENTS[N]` | Nth arg, 0-indexed. Shell-style quoting (wrap multi-word in quotes). |
| `$N` | Shorthand for `$ARGUMENTS[N]` — `$0`, `$1`, etc. |
| `$name` | Named arg from `arguments:` frontmatter. With `arguments: [issue, branch]`, `$issue` = first arg, `$branch` = second. |
| `${CLAUDE_SESSION_ID}` | Session ID — useful for log file naming. |
| `${CLAUDE_EFFORT}` | Current effort (`low`/`medium`/`high`/`xhigh`/`max`). `ultracode` reports as `xhigh`. |
| `${CLAUDE_SKILL_DIR}` | Directory containing this `SKILL.md`. Use this in `!` shell commands to reference bundled scripts — works regardless of CWD or scope. |

Escape with `\$1.00` to keep a literal `$` before a digit / `ARGUMENTS` / declared name. `\$` elsewhere is left unchanged.

**If you invoke with arguments but the body doesn't include `$ARGUMENTS`**, Claude Code appends `ARGUMENTS: <input>` so Claude still sees what you typed.

## Dynamic context injection (`!\`command\``)

Run shell commands **before** Claude sees the skill content. Output replaces the placeholder.

```yaml
---
description: Summarize PR changes
---

## Pull request context
- PR diff: !`gh pr diff`
- Comments: !`gh pr view --comments`

## Task

Summarize this PR.
```

Three things to know:

1. **Pre-processing, not tool use.** Substitution runs once before Claude sees anything. Claude only sees the final rendered text.
2. **Output is not re-scanned.** A command's output cannot emit another `` !`...` `` for a later pass to expand.
3. **`!` recognized only at line start or after whitespace.** `` KEY=!`cmd` `` is left as literal text.

For multi-line commands, use a fenced block opened with ` ```! `:

```
## Environment
\```!
node --version
git status --short
\```
```

`disableSkillShellExecution: true` in settings disables this for user/project/plugin/additional-directory skills (bundled and managed unaffected). Best used as managed policy.

> Including `ultrathink` anywhere in the skill content triggers deeper reasoning when the skill runs.

## Invocation control

| Frontmatter | You can invoke | Claude can invoke | When loaded into context |
| --- | --- | --- | --- |
| (default) | Yes | Yes | Description always in context; body loads on invoke. |
| `disable-model-invocation: true` | Yes | No | Description **not** in context; body loads when you invoke. |
| `user-invocable: false` | No | Yes | Description always in context; body loads when Claude invokes. |

Rule of thumb:
- **`disable-model-invocation: true`** for skills with side effects (`/commit`, `/deploy`, `/send-slack`, `/notify-team`). Don't want Claude deciding to deploy because your code "looked ready."
- **`user-invocable: false`** for background knowledge a user wouldn't meaningfully `/invoke` — e.g. `legacy-system-context` that explains an old subsystem.

## Skill content lifecycle

Once a skill is invoked, its rendered `SKILL.md` enters the conversation as a single message and **stays for the rest of the session**. Claude Code does not re-read the file on later turns.

**Compaction behavior** (also covered in `claude/tools/context-window.md`):
- Most recent invocation of each skill is re-attached after the summary.
- **5,000 tokens per skill, 25,000 tokens total budget.**
- Fills from most-recent-first. Older skills can be dropped entirely if you invoked many.
- Truncation keeps the **start** of the file → put the most important instructions at the top.

If a skill seems to lose influence after the first turn: content is usually still present, the model is choosing other approaches. Strengthen the description and instructions. Or use a hook for deterministic enforcement.

## Pre-approve tools (`allowed-tools`)

Grants permission for listed tools while the skill is active — Claude doesn't prompt. Doesn't restrict (every tool is still callable; non-listed tools follow your normal permission settings).

```yaml
---
name: commit
disable-model-invocation: true
allowed-tools: Bash(git add *) Bash(git commit *) Bash(git status *)
---
```

Project-level `.claude/skills/` `allowed-tools` requires accepting the workspace trust dialog — review project skills before trusting a repo.

`disallowed-tools` removes tools from Claude's pool while skill is active. Clears on your next message. To block tools across all skills, use `permissions.deny`.

## Path-scoped skills (`paths`)

Skill only auto-loads when Claude works with matching files. Same format as path-scoped rules.

```yaml
---
description: Frontend conventions
paths: [src/components/**/*.tsx, src/pages/**/*.tsx]
---
```

Pairs with monorepo nested-skill layout — most efficient context use.

## Run in a subagent (`context: fork`)

The skill content becomes the subagent's prompt. Subagent runs in its own context (no main-session history, separate token budget).

```yaml
---
name: deep-research
description: Research a topic thoroughly
context: fork
agent: Explore
---

Research $ARGUMENTS thoroughly:
1. Find relevant files with Glob and Grep
2. Read and analyze
3. Summarize with file references
```

`agent` options: `Explore`, `Plan`, `general-purpose` (default), or any custom subagent from `.claude/agents/`. Explore and Plan skip `CLAUDE.md` for smaller context.

> `context: fork` only makes sense for skills with **explicit task instructions**. If your skill is just guidelines ("use these API conventions"), the subagent receives them but no task to do → empty output.

## Supporting files

Skills can include any files in their directory. Reference them from `SKILL.md` so Claude knows what each contains and when to load.

```
my-skill/
├── SKILL.md          # Required — overview + navigation
├── reference.md      # Detailed API docs, loaded when needed
├── examples.md       # Loaded when needed
└── scripts/
    └── helper.py     # Executed, not loaded
```

In `SKILL.md`:
```markdown
For complete API details, see [reference.md](reference.md)
For usage examples, see [examples.md](examples.md)
```

Use `${CLAUDE_SKILL_DIR}` in `!` shell commands to reference bundled scripts regardless of CWD or which scope the skill is installed at.

## Bundled skills (ship with Claude Code)

These appear as commands and work like user skills. Disable globally with `disableBundledSkills: true`.

| Skill | What it does |
| --- | --- |
| `/code-review [low/medium/high/xhigh/max/ultra] [--fix] [--comment]` | Review current diff for bugs + reuse/simplify/efficiency cleanups. `--fix` applies findings. `ultra` runs deep cloud review. |
| `/simplify [target]` | Cleanup-only review (no bug hunting). 4 parallel agents: reuse, simplification, efficiency, abstraction level. v2.1.154+. |
| `/debug [description]` | Enable debug logging mid-session and analyze. |
| `/batch <instruction>` | Decompose large change into 5-30 independent units; spawn background subagent per unit in isolated worktree. |
| `/loop [interval] [prompt]` | Repeat a prompt on a schedule. Self-paces if no interval. |
| `/claude-api [migrate|managed-agents-onboard]` | Load Claude API reference for your project's language. Auto-activates on `anthropic` or `@anthropic-ai/sdk` imports. |
| `/run`, `/verify`, `/run-skill-generator` (v2.1.145+) | Launch and drive your app to verify changes. `/run-skill-generator` records the recipe as a per-project skill at `.claude/skills/run-<name>/`. |
| `/fewer-permission-prompts` | Scan transcripts for common read-only Bash + MCP calls; add prioritized allowlist to project `.claude/settings.json`. |

`/run-skill-generator` is the most interesting pattern — it generates a custom skill per project. Run once per project, again if build/launch changes.

## `/skills`, `/reload-skills`, `skillOverrides`

- **`/skills`** — list skills, sort by token count (`t`), hide individual skills (`Space` to cycle states, `Enter` to save to `.claude/settings.local.json`).
- **`/reload-skills`** (v2.1.152+) — re-scan skill + command directories so skills added/changed on disk during the session work without restart.

`skillOverrides` in settings controls per-skill visibility without editing `SKILL.md`:

| Value | Listed to Claude | In `/` menu |
| --- | --- | --- |
| `"on"` | Name + description | Yes |
| `"name-only"` | Name only | Yes |
| `"user-invocable-only"` | Hidden | Yes |
| `"off"` | Hidden | Hidden |

```json
{
  "skillOverrides": {
    "legacy-context": "name-only",
    "deploy": "off"
  }
}
```

Plugin skills aren't affected — manage those through `/plugin`.

## Live change detection

Adding, editing, or removing a skill under `~/.claude/skills/`, project `.claude/skills/`, or a `--add-dir` `.claude/skills/` takes effect within the current session — no restart.

> **Exception**: creating a top-level `.claude/skills/` dir that didn't exist when the session started requires restart so the dir can be watched.

Live change covers `SKILL.md` text only. For skill folders that are also plugins (`.claude-plugin/plugin.json` inside), changes to `hooks/`, `.mcp.json`, `agents/`, `output-styles/` need `/reload-plugins`.

## Skills from additional directories

`--add-dir` / `/add-dir` **load `.claude/skills/`** from the added directory. This is an exception to the "additional directories grant file access only" rule — `additionalDirectories` in settings does **not** load skills.

CLAUDE.md from added dirs only loads if `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1`.

## Restricting Claude's skill access

Three layers:

| Approach | Effect |
| --- | --- |
| `deny: ["Skill"]` in permissions | Disable all skills for Claude. |
| `deny: ["Skill(deploy *)"]` / `allow: ["Skill(commit)"]` | Per-skill control. `Skill(name *)` is prefix match. |
| `disable-model-invocation: true` on the skill | Removes description from Claude's context entirely. |

`user-invocable` only controls menu visibility — not Skill tool access. Use `disable-model-invocation` to block programmatic invocation.

A few built-in commands are also Skill-tool-callable: `/init`, `/review`, `/security-review`. Others like `/compact` are not.

## Skill description budget

Skill descriptions are loaded into Claude's context every turn so it knows what's available. All skill names are always included. **Descriptions are shortened to fit a budget** — at 1% of model context window by default — and dropped from least-used to most-used.

When descriptions are cut, Claude can't match your request to the skill. Diagnostics:

- `/doctor` — see how many descriptions are shortened/dropped and which skills are affected.

Knobs:

| Setting | Effect |
| --- | --- |
| `skillListingBudgetFraction` (e.g. `0.02` = 2%) | Raise the budget. |
| `SLASH_COMMAND_TOOL_CHAR_BUDGET` env var | Fixed character cap. |
| `maxSkillDescriptionChars` (default 1,536) | Per-skill cap. |
| `skillOverrides: { ..: "name-only" }` | Free budget for other skills. |

Trim the source: put the key use case first in `description`. Move overflow to `when_to_use` (still capped together at 1,536).

## Evals (`skill-creator` plugin)

Skills are only useful if Claude actually invokes them on the right prompts *and* the output matches what you expect. Two failure modes; eval both separately.

The `skill-creator` plugin (official Anthropic marketplace) automates the loop:

```
/plugin install skill-creator@claude-plugins-official
```

If "plugin not found":
```
/plugin marketplace update claude-plugins-official
# or
/plugin marketplace add anthropics/claude-plugins-official
```

Then `/reload-plugins` and ask Claude to `evaluate my <name> skill with skill-creator`. Stores test cases in `evals/evals.json` inside the skill dir. Records pass-rate, tokens, time for with-skill vs. without-skill comparisons. Supports A/B between two skill versions, description-tuning loops, and an HTML review viewer.

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| Skill not triggering | Description doesn't include keywords users would naturally say. Try `What skills are available?` to verify Claude sees it. Try rephrasing your request closer to the description. |
| Triggers too often | Make description more specific. Or add `disable-model-invocation: true`. |
| Frontmatter parse error | `claude --debug` shows the YAML error. Skill body still loads with empty metadata — `/skill-name` works but Claude has no description to match against. |
| Descriptions cut short | `/doctor` shows which. Raise `skillListingBudgetFraction`, set low-priority skills to `name-only`, or trim source descriptions. |

## Cross-OS notes

- **Windows shell** in `!` injection: set `shell: powershell` in frontmatter. Requires `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` (unlike hook PowerShell, which doesn't need it — hooks spawn PS directly).
- **Skill discovery is OS-neutral.** Path conventions (`~/.claude/skills/` on Mac, `%USERPROFILE%\.claude\skills\` on Windows) — same as everything else.
- **`${CLAUDE_SKILL_DIR}`** in shell commands is OS-correct on whichever platform the skill runs on.

## Practical applications for this repo

When we start drafting reusable workflows (next phase of the playbook), strong skill candidates:

- **`/playbook-search <topic>`** — `disable-model-invocation: false`, lets Claude grep the repo for related principles. No side effects.
- **`/new-workflow <name>`** — `disable-model-invocation: true`, creates the workflow scaffold. Side effect (file creation).
- **`/cross-os-check`** — auto-loads on edits to setup scripts via `paths:`. Reminds Claude to cover both macOS and Windows.

For the user profile in `~/.claude/`: a personal `/cleanup-branches` skill with `disable-model-invocation: true` and `allowed-tools: Bash(git branch *) Bash(git push *)` would beat memorizing the workflow.

For the Windows VM: skills are not synced (they live in the OS-local `~/.claude/skills/` outside this repo). Consider checking team-wide skills into a private repo and `git pull`-ing on each machine, same approach as private memory.
