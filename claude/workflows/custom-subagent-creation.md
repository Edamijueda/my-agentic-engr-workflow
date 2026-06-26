# Your first custom subagent

How to graduate from built-in `Explore` to a project-specific subagent. **Defer until the rule of three** — see [`../../shared/principles/defer-until-friction.md`](../../shared/principles/defer-until-friction.md).

## When to write one

You should be able to point at three+ past delegations that look the same:

- "Find all places that call X and how they handle errors" (3 times this week, different X each time).
- "Review this PR for our React conventions" (a recurring pattern).
- "Look up our schema for table Y" (every time you touch the DB).

If you can't, you don't have three data points yet. Wait. Use built-in `Explore` / `general-purpose` until the shape of the friction is clear.

## Where to put it

| Scope | Path | When |
| --- | --- | --- |
| **Project** | `.claude/agents/<name>.md` | Project-specific; commit. **Most common.** |
| **Personal** | `~/.claude/agents/<name>.md` | Across all your projects; per-machine. |
| **Plugin** | `<plugin>/agents/<name>.md` | Shareable across teams; install machinery. |

Don't over-think — start with project scope. Move to personal if you find yourself recreating the same agent across projects.

## The starter template

Project subagent at `.claude/agents/api-conventions-checker.md`:

```markdown
---
name: api-conventions-checker
description: Reviews API endpoint implementations for adherence to project conventions. Use after writing or editing API handlers in src/api/.
tools: Read, Grep, Glob
model: inherit
memory: project
---

You are an API convention reviewer for this project.

Check that endpoints:

1. Follow RESTful naming (`/users/:id`, not `/getUser?id=`).
2. Use the standard error format defined in `src/api/errors.ts`.
3. Validate inputs with zod schemas at the route entry point.
4. Include OpenAPI doc comments.

Before reviewing:
- Check your memory for project-specific patterns you've noted before.

During review:
- Cite each violation by file and line.
- Quote the convention being violated.
- Show the fix as a code snippet.

After reviewing:
- Update your memory with any new convention insights you noticed.

Return findings organized by severity:
- Critical (security, data integrity)
- High (convention violations)
- Suggestions (style, optimization)
```

Adapt the body to your domain. The frontmatter shape stays similar.

## Frontmatter cheatsheet

Required:
- `name` — lowercase + hyphens. Drives `/<agent-name>` invocation.
- `description` — drives Claude's automatic delegation. Be specific about *when* to use it.

Most useful optional fields:
- `tools` — allowlist. `Read, Grep, Glob` for read-only research.
- `disallowedTools` — denylist. Inherits all minus these.
- `model: inherit` (default) or `sonnet` / `haiku` / etc.
- `memory: project` — cross-session learning at `.claude/agent-memory/<name>/`.
- `context: fork` — inherit the parent conversation instead of starting fresh.
- `isolation: worktree` — runs in a separate git worktree.
- `hooks:` — lifecycle hooks scoped to this subagent.

Full reference: [`../tools/subagents.md`](../tools/subagents.md) → "Frontmatter reference."

## The `description` is what drives delegation

This is where most custom subagents fail. Claude reads the description to decide when to use the subagent. If it's vague, Claude won't delegate.

Good descriptions:
- "Reviews API endpoint implementations **after writing or editing handlers in `src/api/`**." (Tells Claude *when*.)
- "Audits the test suite **proactively** after code changes. Use when test failures appear." (Includes trigger phrasing.)

Bad descriptions:
- "Code reviewer." (When does Claude invoke this?)
- "Helps with auth." (Too vague.)

> Include phrases like "use proactively after X" or "invoke when Y" — they nudge Claude toward auto-delegation.

## Test the subagent triggers

```
@api-conventions-checker review the changes in src/api/users.ts
```

Forces the subagent to fire. Compare to:

```
Did I follow our API conventions in src/api/users.ts?
```

If Claude doesn't auto-delegate on the second prompt, your description isn't matching the user phrasing. Iterate.

For Claude-Code-specific phrasing patterns, see [`../tools/subagents.md`](../tools/subagents.md) → "Invoke a subagent."

## `memory: project` for cross-session learning

The `memory:` field gives the subagent a persistent directory at `.claude/agent-memory/<name>/` (project) or `~/.claude/agent-memory/<name>/` (user).

What this unlocks:
- The subagent reads its `MEMORY.md` at startup (first 200 lines / 25KB).
- Read/Write/Edit tools auto-enabled so it can manage memory files.
- Over multiple sessions, it accumulates project-specific insights — naming patterns, common bugs, codepath locations.

Tips for memory-enabled subagents:
- Tell the subagent in the body: "Update your memory as you discover patterns."
- Tell the subagent at delegation time: "Check your memory for issues you've seen before."
- Curate `MEMORY.md` periodically — it can grow unfocused.

Full mechanics: [`../tools/subagents.md`](../tools/subagents.md) → "Enable persistent memory."

## Tools — narrow them

The default (no `tools` field) inherits every tool the main conversation has. For most custom subagents, that's too broad.

Common patterns:
- **Read-only research**: `tools: Read, Grep, Glob`. The agent cannot write.
- **Read + Bash for tests**: `tools: Read, Grep, Glob, Bash`. Run tests but can't edit.
- **Full implementer**: `tools: Read, Edit, Write, Bash, Grep, Glob`. For agents that finish tasks.

Narrowing `tools`:
- Protects you (subagent can't take unintended actions).
- Reduces the subagent's available-tool list — less context cost.
- Forces clearer interface (return summary, not random tool calls).

## Hooks for guardrails (optional)

For "this subagent must never X" patterns, attach a hook in frontmatter:

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "${CLAUDE_PROJECT_DIR}/.claude/hooks/no-prod-deploys.sh"
```

The hook script reads JSON via stdin, exits 2 to block. Useful when you want the subagent to have Bash but not be able to run specific commands.

Full mechanics: [`../tools/hooks.md`](../tools/hooks.md) and [`../tools/subagents.md`](../tools/subagents.md) → "Hooks in frontmatter."

## Common mistakes

- **Writing it on day one.** Defer until friction. See [`../../shared/principles/defer-until-friction.md`](../../shared/principles/defer-until-friction.md).
- **Description too vague.** Claude won't auto-delegate. Iterate based on what phrasing you naturally use.
- **Tools too broad.** Defeats the isolation. Narrow to what the subagent actually needs.
- **Loading the prompt with everything you know.** Subagents inherit project `CLAUDE.md` (except Explore and Plan). Don't duplicate.
- **Forgetting `memory: project`.** The whole point of a custom subagent for a recurring task is that it learns. Without memory, every invocation is fresh.
- **Naming it generically.** `code-reviewer` is fine for the agent's job; `react-component-reviewer-for-this-project` is more discoverable in `/agents`.

## Manage with `/agents`

`/agents` opens a tabbed UI:
- **Running** — live + recently finished subagents. Open or stop them.
- **Library** — all subagents (built-in, user, project, plugin). Create, edit, delete. Shows which is active on name collisions.

Faster than editing the file directly when you're tuning.

## See also

- [`../tools/subagents.md`](../tools/subagents.md) — full subagent reference.
- [`scaffolding-decision.md`](scaffolding-decision.md) — when a subagent is the right answer vs. a skill, hook, or memory entry.
- [`memory-curation.md`](memory-curation.md) — companion (the agent's memory) and the parent's (CLAUDE.md) deserve different curation rhythms.
- [`../../shared/principles/defer-until-friction.md`](../../shared/principles/defer-until-friction.md) — when to write any custom scaffolding.
