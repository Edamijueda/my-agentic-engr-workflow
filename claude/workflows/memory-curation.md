# Memory curation

`CLAUDE.md` isn't write-once. It curates as the project evolves and as you learn what works.

## When CLAUDE.md needs work

Triggers worth acting on:

- **"Why does Claude not know X about this codebase?"** — you've been re-explaining a fact. Add the fact.
- **Same correction comes up 3 times** — codify the rule (see [`scaffolding-decision.md`](scaffolding-decision.md) for whether it belongs in CLAUDE.md or elsewhere).
- **A teammate joining can't get productive without explanation** — the explanation belongs in CLAUDE.md.
- **File getting long (>200 lines)** — adherence drops; time to split or move things to `.claude/rules/` with `paths:`.
- **Contradictions** — two rules disagree. Audit and pick.
- **After major refactors** — the build commands, file layout, or conventions captured at `/init` may be stale.

## What belongs in CLAUDE.md

| | Belongs in CLAUDE.md |
| --- | --- |
| ✅ | Build / test / lint commands. |
| ✅ | Project-wide naming and code style. |
| ✅ | Always-true facts about file layout the agent can't derive. |
| ✅ | "Always do X" / "Never do Y" rules. |
| ✅ | Links to design docs the agent should consult. |
| ❌ | Things derivable from reading code (file paths, type signatures, function bodies). |
| ❌ | One-off task context (that belongs in the conversation). |
| ❌ | Multi-step procedures (move to a skill). |
| ❌ | Rules that only apply in one area (move to `.claude/rules/` with `paths:`). |
| ❌ | Hard "must never" guarantees (use `permissions.deny` or a hook). |

## Splitting strategies

### Strategy 1: Move area-specific rules to `.claude/rules/`

When `CLAUDE.md` grows past ~200 lines and you can identify chunks that only matter in part of the codebase:

```markdown
---
# .claude/rules/api.md
paths:
  - "src/api/**"
---

# API conventions

- Use the standard error format in src/api/errors.ts
- Validate inputs with zod schemas at the route boundary
- ...
```

These rules **only load when Claude touches matching files**. Saves context every session you're NOT working in `src/api/`.

**Gotcha**: rules with `paths:` frontmatter are **summarized away by `/compact`** and only reload the next time Claude reads a matching file. If the rule must survive compaction (rare), drop the `paths:` or move it to CLAUDE.md.

Full mechanics: [`../config/memory.md`](../config/memory.md) → "Path-scoped rules."

### Strategy 2: Use `@imports`

For organizational splitting (still loads in full at session start):

```markdown
# CLAUDE.md

This is a TypeScript monorepo with a React frontend and a Bun backend.

@docs/conventions.md
@docs/architecture.md
```

Imports recurse up to 4 levels. Use this when you want clean organization but the content is always-on. **Doesn't save context** the way `paths:` rules do.

### Strategy 3: Move procedures to skills

If `CLAUDE.md` is starting to read like a how-to guide ("To deploy: 1. Run X. 2. Run Y. 3. ..."), it's a skill, not memory. Move it to `.claude/skills/<name>/SKILL.md` and reference from CLAUDE.md if needed:

```markdown
# CLAUDE.md
...
For deployment, see the `/deploy` skill.
```

The skill body loads only when invoked.

## Regenerate with `/init`

After major refactors (or just periodically), run `/init` again. It:
- Reads the current codebase.
- Suggests improvements rather than overwriting existing `CLAUDE.md`.
- Catches things like stale build commands or moved file paths.

Set `CLAUDE_CODE_NEW_INIT=1` for the interactive multi-phase flow that also walks through skills, hooks, and personal memory files.

## Auto-memory complement

Two memory systems:

| | CLAUDE.md (you write) | Auto memory (Claude writes) |
| --- | --- | --- |
| **What** | Instructions, conventions, facts | Claude's notes from prior sessions |
| **Reload** | Re-injected from disk on every session + after `/compact` | First 200 lines / 25KB of MEMORY.md re-injected after `/compact` |
| **Location** | `CLAUDE.md` at repo root | `~/.claude/projects/<project>/memory/` (machine-local) |
| **Scope** | Cross-machine if committed | Machine-local |

Auto-memory is Claude's own notes — build commands it learned, debugging insights, recurring issues. Look at it occasionally with `/memory` to see what Claude has saved.

For things you want to follow you to your Windows VM, put them in **`CLAUDE.md`** (committed). Auto-memory does not cross machines.

## Quality checks

Run periodically (every few weeks on an active project):

- **Can the agent derive this from the code?** → Remove.
- **Does this apply to all of the codebase?** → If no, move to `.claude/rules/` with `paths:`.
- **Has this rule been ignored 3+ times?** → Strengthen wording, or **escalate to a hook** ([`scaffolding-decision.md`](scaffolding-decision.md)).
- **Does this rule contradict another rule?** → Audit and pick.
- **Has this rule grown into a procedure?** → Move to a skill.
- **Is the file over 200 lines?** → Split.

## Common mistakes

- **Letting CLAUDE.md grow past 200 lines** without splitting. Adherence drops alongside the token cost.
- **Writing instructions for a single current task** instead of always-true rules. Use the conversation for one-offs.
- **Repeating things visible in the code.** "We have an `auth.ts` file" — Claude reads the directory. Don't restate.
- **Not running `/init` after major refactors.** Stale build commands and file paths confuse the agent.
- **Treating CLAUDE.md as the enforcement layer.** It's influence. For enforcement, use `permissions.deny` or hooks (see [`../../shared/principles/enforcement-vs-influence.md`](../../shared/principles/enforcement-vs-influence.md)).
- **Editing CLAUDE.md mid-session and expecting changes to apply.** They don't — `CLAUDE.md` is loaded once at session start. Changes take effect after `/clear`, `/compact`, or restart. See [`../tools/prompt-caching.md`](../tools/prompt-caching.md).

## What survives `/compact`

Quick reference (full table in [`../tools/context-window.md`](../tools/context-window.md)):

| Survives `/compact`? | What |
| --- | --- |
| ✅ Re-injected from disk | Project-root `CLAUDE.md`, unscoped `.claude/rules/`, auto memory |
| ❌ Lost until matching file re-read | `.claude/rules/` with `paths:` frontmatter, nested `CLAUDE.md` |

This affects splitting strategy. Path-scoped rules are great for context savings but disappear after `/compact` until a matching file is re-read. For rules that must always be in effect, prefer the root `CLAUDE.md`.

## See also

- [`../config/memory.md`](../config/memory.md) — full mechanics for CLAUDE.md, rules, auto-memory.
- [`scaffolding-decision.md`](scaffolding-decision.md) — when memory is the right answer vs. skills / hooks / permissions.
- [`../tools/context-window.md`](../tools/context-window.md) — what survives `/compact`.
- [`../tools/prompt-caching.md`](../tools/prompt-caching.md) — why mid-session edits don't apply.
- [`../../shared/principles/enforcement-vs-influence.md`](../../shared/principles/enforcement-vs-influence.md) — memory is influence, not enforcement.
