# Starting a new project — checklist

What to do when you open a coding agent on a project for the first time. Agent-agnostic at the top; Claude-specific implementation links out to `claude/`.

## TL;DR — the three things

If you do nothing else, do these:

1. **Write a project memory file.** For Claude Code: `CLAUDE.md` at the repo root. For other agents: their equivalent. Keep it under 200 lines. State conventions, build/test commands, and "always do X" rules.
2. **Block secrets and high-blast-radius commands.** Deny reads of `.env*`, secrets dirs, and credentials. Deny direct internet shell commands (`curl http*`, `wget *`). Deny force-pushes to `main`.
3. **Default to plan mode (or your agent's equivalent).** Make the agent describe what it will do before it edits. This is the single biggest "stay in control" lever.

Everything below is depth on these three plus optional scaffolding to consider once you're past the first session.

---

## Step 1: Project memory

The file that gets loaded into the agent's context every session. The agent uses it to know what's true about this project.

**For Claude Code**: `CLAUDE.md` at the repo root. Full mechanics in [`claude/config/memory.md`](../../claude/config/memory.md) — scope ladder (managed > user > project > local), `.claude/rules/` for path-scoped instructions, imports via `@path`, `/init` command.

### What belongs there

- **Build + test commands.** Specific, copy-pasteable: `npm run build`, `pytest tests/`, `make lint`.
- **Project layout** that isn't obvious from the file tree (e.g. "API handlers live in `src/api/handlers/`", "shared utilities in `packages/core/`").
- **Conventions** that affect every edit: indentation, naming, comment style, error handling philosophy.
- **"Always do X" / "Never do Y"** rules that came from past corrections.
- **External references** the agent should consult: design doc URL, schema location, runbook.

### What does NOT belong

- **Things the agent can derive from the code.** File paths, function signatures, type definitions — agents read these directly.
- **One-off task context.** That's for the conversation, not memory.
- **Multi-step procedures.** Those belong in skills (Claude Code: [`claude/tools/skills.md`](../../claude/tools/skills.md)) or a Makefile.
- **Path-specific rules.** If a rule only applies in `src/api/`, scope it (Claude Code: `.claude/rules/api.md` with `paths:` frontmatter). The agent only loads it when touching matching files.
- **Hard "must / must not" guarantees.** Memory is influence, not enforcement. Use permissions or hooks for hard guarantees.

### Quick checks

- **Under 200 lines.** Longer files dilute attention.
- **No conflicting instructions.** Audit periodically; agents may pick arbitrarily when rules contradict.
- **Concrete over abstract.** "Use 2-space indentation" beats "format code properly". "Run `npm test` before committing" beats "test your changes".

> **For Claude Code specifically**: run `/init` once to generate a starter from the codebase. Refine from there with things the agent wouldn't discover on its own. Set `CLAUDE_CODE_NEW_INIT=1` for the interactive multi-phase flow that walks through skills, hooks, and personal memory too.

---

## Step 2: Permissions baseline

Deny things the agent should never do, regardless of how reasonable the request sounds in conversation. This is the "stay in control" layer.

**For Claude Code**: `.claude/settings.json` (committed to git). Full mechanics in [`claude/config/permissions.md`](../../claude/config/permissions.md) — first-match-wins evaluation (deny > ask > allow), bare-name vs scoped denies, Read/Edit anchor traps.

### The starter deny block

```json
{
  "permissions": {
    "defaultMode": "default",
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(./config/credentials.json)",
      "Bash(curl http*)",
      "Bash(wget *)",
      "Bash(git push * main)",
      "Bash(git push * master)",
      "Bash(git push --force *)",
      "Bash(git push -f *)"
    ],
    "ask": [
      "Bash(rm -rf *)",
      "Bash(npm publish *)"
    ]
  }
}
```

Adapt to the project:
- **More secrets paths**: any file containing tokens, API keys, or `.env`-like content.
- **Risky external services**: `Bash(aws s3 rm *)`, `Bash(gcloud * delete *)`, `Bash(supabase * delete *)`, `Bash(terraform destroy *)`, etc.
- **Production branch protection**: replace `main`/`master` with your project's protected branch.

### Personal overrides (not committed)

`.claude/settings.local.json` for personal allows that shouldn't be team-shared (e.g. project-specific Bash commands you use a lot):

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git status)"
    ]
  }
}
```

Gets gitignored automatically when Claude Code creates it.

### Hard "must not" — go beyond permissions

Permissions are first-match-wins and have a few edge cases (process wrappers, dev-env runners). For absolute "this must never happen", use a hook (Claude Code: [`claude/tools/hooks.md`](../../claude/tools/hooks.md)) — `PreToolUse` exiting code 2 blocks before permission rules are evaluated. Example: a hook that rejects any `git push` to `main` regardless of allow rules elsewhere.

### Cross-OS gotcha

- On **native Windows**, there's **no sandbox** (Seatbelt/bubblewrap aren't available). Compensate with stricter `permissions.deny` rules on the Windows machine, or switch to WSL 2 for sandboxed development.

---

## Step 3: Plan-mode-first default

Plan mode = the agent explores and proposes a plan but doesn't edit. You approve, refine, or redirect before any code changes. This is where "code captain" is won or lost.

**For Claude Code**: set `permissions.defaultMode: "plan"` in **`~/.claude/settings.json`** (your personal default — *not* `.claude/settings.json`, since auto-mode and similar values are intentionally ignored from in-repo settings as a repo-spoof guard).

Why your personal default is the right place:
- Plan is **not auto-promoting** — you still control when to accept a plan.
- Some projects (small experiments, prototypes) genuinely don't need plan mode. Per-project override works fine.
- A team default of `acceptEdits` in `.claude/settings.json` would override your personal plan setting at the project level — you'd lose the habit. Personal scope is robust.

### What plan mode shows you

For Claude Code: a plan markdown document with sections, opens for editing before approval. `Ctrl+G` to edit in your default text editor. On accept, prompts for the next mode (`auto`, `acceptEdits`, or `default`).

### When to skip plan mode

- **One-line fixes.** Typo, comment, obvious config tweak.
- **Pure read tasks.** "What does this function do?" doesn't need a plan.
- **Sessions where you're driving step-by-step**, not delegating big chunks.

Switch back to plan mode with `Shift+Tab` whenever the next prompt is non-trivial. The friction of toggling is the point — it's a checkpoint.

---

## Optional scaffolding (defer unless you have a reason)

Don't add these on day one. Add them when the friction shows up:

| Add when… | Use |
| --- | --- |
| You keep typing the same correction into chat | Update `CLAUDE.md` (memory). |
| You keep invoking the same multi-step procedure | Create a skill ([`claude/tools/skills.md`](../../claude/tools/skills.md)). `disable-model-invocation: true` for side-effect-y ones (`/deploy`, `/commit-push`). |
| You need a procedure to fire automatically at a specific moment | Hook ([`claude/tools/hooks.md`](../../claude/tools/hooks.md)). `PreToolUse` for guards, `PostToolUse` for auto-format, `SessionStart` for context injection. |
| A side task floods your conversation with logs / search output | Delegate to a subagent ([`claude/tools/subagents.md`](../../claude/tools/subagents.md)). Built-in `Explore` is read-only and fast — good default for research. |
| You're starting a long task that needs sustained focus | `/plan` for the plan, then accept into `acceptEdits`. Use a subagent for any verbose sub-step. |
| The project uses an external system (Jira, Drive, Slack) | MCP server (deferred doc; see [Anthropic's MCP docs](https://code.claude.com/docs/en/mcp)). |
| Personal preferences across all projects | `~/.claude/CLAUDE.md` (not the project file). |

> **The defer-until-friction rule matters.** Premature scaffolding is wasted context and wasted setup time. Most projects need exactly: `CLAUDE.md`, `.claude/settings.json` with denies, and personal plan-mode default. That's it for day one.

---

## Pre-flight checklist (commands in order)

The literal sequence for a fresh project, assuming Claude Code is already installed and authenticated (if not, see [`claude/setup/install.md`](../../claude/setup/install.md) and [`claude/setup/authentication.md`](../../claude/setup/authentication.md)).

```bash
# 1. cd into the new project, open it in Claude Code
cd path/to/project
claude
```

In the session:

```
# 2. Generate a starter CLAUDE.md
/init
```

Review the generated `CLAUDE.md`. Trim or rewrite where it guessed wrong. Aim for under 200 lines.

```
# 3. Open settings to add the deny block
/permissions
```

Or edit `.claude/settings.json` directly. Paste the starter deny block above; adapt to project risks.

```
# 4. Check what's loaded
/status
/context
```

`/status` confirms which settings sources loaded (look for project + user). `/context` shows where the token budget is going at startup — if `CLAUDE.md` is eating more than ~2K tokens, trim.

```
# 5. Verify plan-mode default is your personal default
cat ~/.claude/settings.json   # macOS/Linux
type %USERPROFILE%\.claude\settings.json   # Windows CMD
```

Should include `"permissions": { "defaultMode": "plan" }`. If not, add it (see Step 3 above).

```
# 6. (Optional) Commit the project-scope files
git add CLAUDE.md .claude/settings.json
git commit -m "Add Claude Code project memory and permissions"
```

`.claude/settings.local.json` is auto-gitignored when Claude Code creates it. If you created it manually, gitignore it.

### Quick verification

Open a new session in the project and ask:
```
What conventions does this project use, and which permissions are denied?
```

Claude should answer from `CLAUDE.md` and reference the deny rules without you re-explaining. If it can't, your memory or permissions aren't loading — `/doctor` will surface the issue.

---

## Cross-OS notes

Most steps are OS-neutral. The OS-specific bits:

- **Permission rule paths**: Read/Edit rules use POSIX form on both OSes. Windows `C:\Users\alice\.env` normalizes to `/c/Users/alice/.env` — use `//c/**/.env` to match anywhere on C: drive. See [`claude/config/permissions.md`](../../claude/config/permissions.md).
- **Personal settings path**: `~/.claude/settings.json` on Mac/Linux, `%USERPROFILE%\.claude\settings.json` on Windows. Same content.
- **Sandbox availability**: macOS (Seatbelt) + Linux/WSL2 (bubblewrap) — yes. Native Windows — no. On native Windows, lean harder on `permissions.deny`.
- **Bash tool on Windows**: needs Git for Windows for the Bash tool; otherwise falls back to PowerShell. See [`claude/setup/install.md`](../../claude/setup/install.md).
- **The credentials file** lives in different places per OS (macOS Keychain, Linux `~/.claude/.credentials.json`, Windows `%USERPROFILE%\.claude\.credentials.json`). Per-machine — don't try to copy across.

---

## What to skip on day one

Easy to over-engineer first-session setup. Skip until proven necessary:

- **Custom subagents.** Built-in `Explore` and `Plan` cover most research needs. Write a custom subagent when you've delegated the same kind of task 3+ times.
- **Hooks.** Add when memory is being ignored on a critical rule, not preemptively.
- **MCP servers.** Add when the project needs Jira/Drive/etc., not as a generic "agent should have all tools".
- **`auto` mode.** Powerful but a research preview; the classifier blocks many git-rewriting commands by default. Use `plan` or `acceptEdits` until you understand what auto blocks in your workflow.
- **Plugins.** Many are useful but each adds context and complexity. Add `skill-creator` if you start writing custom skills; otherwise wait.

---

## See also

- [`claude/setup/`](../../claude/setup/) — install, auth, VS Code extension. Run through these first if you're on a fresh machine.
- [`claude/config/memory.md`](../../claude/config/memory.md), [`claude/config/settings.md`](../../claude/config/settings.md), [`claude/config/permissions.md`](../../claude/config/permissions.md) — full mechanics for the three foundational config layers.
- [`claude/slash-commands/built-in.md`](../../claude/slash-commands/built-in.md) — the `/init` / `/memory` / `/permissions` / `/status` / `/context` / `/doctor` reference.
- [`claude/tools/context-window.md`](../../claude/tools/context-window.md) — what `/context` is showing you and how to reduce usage.
