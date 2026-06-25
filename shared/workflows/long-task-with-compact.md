# Long task with `/compact`

Token-budget management for sessions that run an hour or more. The four levers: `/context`, `/compact`, `/clear`, `/rewind`.

## When this matters

- You've been working in the same session long enough to feel latency growing.
- You're switching to an unrelated task in the same session.
- You went down a wrong path and want to back out.
- The status bar shows the context window approaching full.

Auto-compaction handles the absolute worst case — but it picks what to keep, not you. Compact yourself at natural breaks instead.

## The four levers

| Lever | What it does | Cost |
| --- | --- | --- |
| **`/context`** | Diagnostic — colored grid showing where the budget is going. | Free. Run anytime. |
| **`/compact <focus>`** | Summarizes conversation history with focus instructions; **startup content reloads** (CLAUDE.md, memory). | Generates a summary (one extra model call). Conversation cache rebuilds afterward. |
| **`/clear`** | Empty conversation, keep startup content loaded. | Faster than `/compact` (no summarization). Loses all conversation. |
| **`/rewind`** | Truncate back to an earlier turn. **Preserves prompt cache** — the prefix still matches. | Cheapest; just truncation. |

## Decision tree

```
What do you want to do?

├── Get rid of unrelated history but keep the current direction
│   → /compact focus on <current direction>
│
├── Abandon what you've done and go back to working state
│   → /rewind to before the bad turn  (preserves cache)
│
├── Switch to a completely unrelated task
│   → /clear  (loses conversation; cheaper than /compact)
│
└── Don't know yet — diagnose first
    → /context  (then pick one of the above)
```

## Step-by-step strategies

### Strategy 1: Compact at natural breaks (proactive)

Best for long tasks that you'll keep working on:

1. **At the end of a coherent phase** (after planning, after the first implementation pass, before testing), run `/context` to see budget usage.
2. If you're past ~50% used, run `/compact focus on <next phase>`. Examples:
   - `/compact focus on the auth bug fix and the failing tests`
   - `/compact focus on the refactor approach we agreed on`
3. Continue working. Conversation cache rebuilds — first turn after compact is slower (full prefix recompute), subsequent turns are cached normally.

### Strategy 2: Rewind a bad path (corrective)

Best for "I went the wrong direction":

1. `/rewind`
2. Pick the message to roll back to (the last point where you were still on the right track).
3. Choose:
   - **Rewind code to here** — revert files, keep conversation. Lets you re-explain from a clean working state.
   - **Fork from here** — new branch, keep code. Lets you try a different approach without losing the first one.
   - **Fork and rewind code** — both.
4. Resume work with corrected direction.

> **`/rewind` is much cheaper than `/compact`** for this case. The prompt cache still matches the prefix up to where you rewound to — no recompute. `/compact` would force a full recompute on the next turn.

### Strategy 3: Clear between unrelated tasks (boundary)

Best for "I just finished X, now switching to Y":

1. Verify X is committed or saved.
2. `/clear`
3. Start the new task fresh. CLAUDE.md and memory reload; conversation is empty.

> **Don't `/clear` when you'll need the conversation context later.** Use `/compact` instead — keeps the gist while freeing space.

## Auto-compaction (the fallback)

`autoCompactEnabled: true` (default) — when context is about to fill, Claude Code auto-compacts. The auto pass picks what to summarize on its own.

**Why prefer manual:**
- You control the focus instructions.
- You control the timing (between tasks vs mid-task).
- Less surprise — the auto pass interrupts mid-thought.

**Don't disable auto-compaction.** It's the safety net for when you forget. Just compact yourself first when you can.

## What survives compaction

Quick reference (full table in [`claude/tools/context-window.md`](../../claude/tools/context-window.md)):

| Survives `/compact`? | What |
| --- | --- |
| ✅ Re-injected from disk | Project-root `CLAUDE.md`, unscoped `.claude/rules/`, auto memory |
| ✅ Re-injected (with cap) | Invoked skill bodies (5K each, 25K total budget, oldest dropped) |
| ✅ Unchanged | System prompt, output style |
| ❌ Lost until matching file re-read | Path-scoped rules with `paths:` frontmatter, nested `CLAUDE.md` |
| ❌ Becomes a summary | Conversation history, file contents, command output |

> **Path-scoped rules disappear after `/compact`** until the matching file is re-read. If a rule must persist through compaction, drop the `paths:` frontmatter or move it to project-root `CLAUDE.md`.

## Common mistakes

- **Letting auto-compact fire mid-task.** Compact yourself between tasks, not during.
- **`/clear` when you'll need the conversation later.** `/compact` keeps the gist.
- **`/compact` to abandon a wrong path.** `/rewind` is the right tool — cheaper, preserves cache.
- **Switching `/model` or `/effort` mid-task.** Both invalidate the prompt cache; next turn fully recomputes. Pick once at session start. See [`claude/tools/prompt-caching.md`](../../claude/tools/prompt-caching.md).
- **Editing `CLAUDE.md` mid-session and expecting changes to apply.** They don't — `CLAUDE.md` is loaded once at session start. Need `/clear` or restart to pick up changes. (Cache stays intact in the meantime, which is the catch.)

## Prevention is cheaper

Compact-when-needed is reactive. Things that reduce *need* to compact:

- **Delegate research to subagents** ([`research-then-edit.md`](research-then-edit.md)). File reads stay in their context, not yours.
- **Keep CLAUDE.md tight** (<200 lines). Big memory files burn budget every turn.
- **Use path-scoped rules** for area-specific guidance — loads only when relevant.
- **`disable-model-invocation: true`** on side-effect skills. Description doesn't appear in skill listing; zero context cost until `/invoke`d.
- **Specific prompts.** "Fix the bug in `auth.ts` line 42" beats "fix the auth bug." Stops the agent reading three files instead of one.

See [`../principles/context-is-finite.md`](../principles/context-is-finite.md) for the full lever set.

## Per-tool implementation

For Claude Code:
- **`/context [all]`** — visualize current usage. `all` expands per-item breakdown.
- **`/compact [focus instructions]`** — manual compact. Focus instructions optional but recommended.
- **`/clear [name]`** — new conversation. Optional name labels the cleared session in `/resume`.
- **`/rewind`** — aliases `/checkpoint`, `/undo`. Hover any message to reveal the rewind button.
- **`autoCompactEnabled`** setting toggles auto-compaction. Leave on.

Full mechanics: [`claude/tools/context-window.md`](../../claude/tools/context-window.md).

## See also

- [`research-then-edit.md`](research-then-edit.md) — the main lever for *not needing* compact as often.
- [`diff-review-cycle.md`](diff-review-cycle.md) — `/rewind` overlap (escape from a wrong path during review).
- [`../principles/context-is-finite.md`](../principles/context-is-finite.md) — the underlying principle.
- [`claude/tools/prompt-caching.md`](../../claude/tools/prompt-caching.md) — why `/rewind` beats `/compact` on cost.
