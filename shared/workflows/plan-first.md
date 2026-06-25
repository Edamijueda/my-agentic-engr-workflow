# Plan-first

The agent describes what it will do **before** it edits. You approve, refine, or redirect. The single biggest "stay in control" lever.

## When it earns its overhead

- Multi-file changes.
- Refactors of any size.
- Any task where you couldn't trivially describe all the changes upfront.
- Unfamiliar codebase or unfamiliar module.
- Risk of touching the wrong code.

## When to skip it

- One-line typo or comment fix.
- Pure read tasks ("what does this function do?").
- Single-file obvious change.
- You're driving step-by-step and approving each edit anyway.

If you're not sure, plan. The friction of approving the plan is the point — it's a checkpoint.

## The pattern

1. **Enter plan mode.** `/plan` in chat (or with the task: `/plan fix the auth bug`). Or use `Shift+Tab` to cycle. Or set `permissions.defaultMode: "plan"` in `~/.claude/settings.json` so plan mode is your personal default (see [`new-machine.md`](../project-setup/new-machine.md)).
2. **State the task.** Be specific. The agent's plan is only as good as the framing.
3. **Let the agent explore.** It reads files and runs read-only shell commands; no edits happen. The exploration goes into the conversation but stays read-only.
4. **Review the plan.** When the agent presents it, read it carefully. Use `Ctrl+G` (terminal) to open the proposed plan in your default editor and modify it directly before approving.
5. **Refine if needed.** Tell the agent what to change: "skip the migration step", "the renderer also needs updating", "use the existing utility in `src/utils/auth.ts`". The agent re-plans.
6. **Accept into the right mode.** The accept-into menu offers:
   - **default** — review each edit individually. Most cautious.
   - **acceptEdits** — agent edits, you review after. The most common choice for non-prod work.
   - **auto** — classifier supervises tool calls (research preview). Locked-down environments only.

## Accept-into decision

| Situation | Accept into |
| --- | --- |
| Pre-production, you'll review the diff anyway | `acceptEdits` |
| Touching sensitive code (auth, payments, prod config) | `default` (review each edit) |
| Long mechanical task where you trust the plan and have your deny rules | `auto` (if available) |
| A teammate's PR; you're driving from review feedback | `default` |

> **Default to `acceptEdits` until a task feels risky enough to want per-edit review.** Plan mode already gave you a chance to course-correct; per-edit review during execution is rarely additive.

## Pre-task tips

- **`/clear` first if the session is stale.** Plan quality drops when the conversation is full of unrelated history.
- **Specify the entry point.** "Refactor the auth flow" is fine; "refactor the auth flow starting from `src/api/auth/refresh.ts`" is better.
- **Mention non-obvious constraints.** "Don't change the public API," "skip the migration to MV3 for now," etc.

## During planning

- The agent reads files freely — see [`claude/config/permissions.md`](../../claude/config/permissions.md) for what's read-only by default.
- The plan markdown can be pretty long. The size isn't the cost; the cost was the file reads. **Don't truncate your review** because the plan looks big.
- If the agent is hitting too many files, it's probably the wrong agent for the task. For deep research, use [`research-then-edit.md`](research-then-edit.md) — delegate exploration to a subagent.

## After approval

- Watch the first 2-3 edits to confirm the agent is following the plan, not improvising.
- If the agent diverges, `Esc` to stop and ask why. Don't let it improvise far before you check.
- If the plan turns out to be wrong mid-execution, `/rewind` is cheaper than `/compact` — see [`long-task-with-compact.md`](long-task-with-compact.md).

## Avoid

- **Accepting a plan you haven't read.** The whole point is the checkpoint. If you're not going to read it, skip plan mode.
- **Accepting into `auto` mode without trust in your deny rules + the classifier's behavior on your specific git workflow.** Auto blocks several git-rewriting commands by default — check [`claude/config/permissions.md`](../../claude/config/permissions.md) for the list.
- **Re-planning the same task type session after session.** That's a signal to write a skill (see [`shared/principles/defer-until-friction.md`](../principles/defer-until-friction.md) — rule of three).

## Per-tool implementation

For Claude Code specifically:
- `/plan [description]` enters plan mode, optionally with the task included.
- `Shift+Tab` cycles through `default` → `acceptEdits` → `plan`.
- `Ctrl+G` opens the proposed plan in your default editor.
- `showClearContextOnPlanAccept: true` setting offers to clear planning context first on each accept option.

Full mechanics: [`claude/config/permissions.md`](../../claude/config/permissions.md) → "Permission modes (the session-level posture)".

## See also

- [`research-then-edit.md`](research-then-edit.md) — when plan mode's reads still feel heavy, push exploration into a subagent.
- [`diff-review-cycle.md`](diff-review-cycle.md) — the review loop after a plan has executed.
- [`../principles/enforcement-vs-influence.md`](../principles/enforcement-vs-influence.md) — plan-mode-default in user scope, why.
