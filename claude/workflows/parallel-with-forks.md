# Parallel approaches with `/fork`

Try multiple approaches from the same starting point. Forks **share the parent's prompt cache** — cheap. Each fork returns a summary back to the main conversation; you compare and pick.

## The mechanic (Claude-specific)

A fork is a subagent that **inherits the entire parent conversation** — same system prompt, tools, model, message history. The only thing it doesn't inherit is *output*: its tool calls stay in the fork's transcript, only the final result returns to main.

**Why it's cheap**: a fork's first request reuses the parent's prompt cache because the prefix matches. A fresh subagent has its own cache (separate). For tasks needing the same context, forks are several times cheaper to spin up than fresh subagents.

Full mechanics: [`claude/tools/subagents.md`](../tools/subagents.md) → "Forks" section.

## When to use it

- **"Try approach A and approach B"** — refactor one way, then fork to try the other.
- **A/B exploration where the starting point matters** — fork preserves your conversation, exploration, and any plan you've already approved.
- **Independent variations of the same task** — different framework migration targets, different naming conventions, etc.
- **You want to compare without losing the first attempt** — forks keep both alive until you dismiss.

## When NOT to use it

- **Linear single-path task** — just do it. Forking adds overhead.
- **Need to compare results live as they execute** — forks return summaries, not running output.
- **You'd need nested forks** — forks cannot spawn forks (they can spawn other subagent types).
- **Heavy file-edit conflicts** — forks can pass `isolation: "worktree"` to edit in a separate worktree, but watch out when forks AND main both want to edit the same files.

## The pattern

1. **Get the main session to a known-good baseline.** Plan reviewed, exploration done. The fork inherits everything; clutter inherits with it.
2. **Spawn forks** — `/fork <directive>` with a clear task per fork:
   ```
   /fork refactor src/auth/ to use the session cookie approach
   /fork refactor src/auth/ to use the JWT approach
   ```
3. **Continue in main** while forks run. Or wait. Up to you.
4. **Each fork returns** as a message in main when it finishes. The panel below your prompt shows progress.
5. **Compare results** in main. The forks themselves remain available — you can dismiss the ones you don't want.
6. **Pick a winner** and continue with that approach in main. (Or pick what you actually want and prompt main to implement it.)

## The fork panel (controls)

When forks are running:

| Key | Action |
| --- | --- |
| `↑` / `↓` | Move between rows |
| `Enter` | Open the selected fork's transcript + send follow-up messages |
| `x` | Dismiss a finished fork or stop a running one |
| `Esc` | Return focus to the main prompt input |

## Variations

### Forks with worktree isolation

When you spawn a fork via the Agent tool, Claude can pass `isolation: "worktree"`. Each fork edits in its own git worktree under `.claude/worktrees/`. Main's checkout stays untouched.

Pair with `/batch` for large-scale parallelism: decompose a change into 5-30 units, one background subagent per unit in an isolated worktree. See [`claude/slash-commands/built-in.md`](../slash-commands/built-in.md).

### Forks for test sweeps

Fork once to run the test suite while you continue working in main:

```
/fork run the full test suite and tell me which tests fail and the root cause
```

Fork returns a summary; you keep coding.

## Enabling fork mode

`CLAUDE_CODE_FORK_SUBAGENT=1` enables explicitly (interactive, headless, SDK). `=0` disables everywhere. Default-on from v2.1.161+.

When fork mode is enabled, **every subagent spawn runs in background**, not just forks. Set `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` for synchronous spawns instead.

## Common mistakes

- **Forking from a session with stale clutter.** The fork inherits everything. `/compact focus on <current direction>` first if the conversation is heavy.
- **Forking + editing in main simultaneously.** Race on file state. Either let the fork finish first, or use `isolation: "worktree"` so they edit independently.
- **Trying to nest forks.** Forks can't spawn forks. They can spawn other subagent types — those count toward the depth-5 cap.
- **Spawning many forks at once.** Each fork's final summary lands in main on completion. Five summaries returning in quick succession can crowd main worse than just picking one approach.
- **Forgetting to dismiss losing forks.** They show in the panel until dismissed (`x`).

## Cache + permission notes

- **Prompt cache shared with main.** First fork request is cache-cheap; subsequent are normally cached. Switching `/model` or `/effort` in the fork would invalidate.
- **Permission prompts surface in your main terminal** while forks run (v2.1.186+), naming which fork is asking. Esc denies one tool call without stopping the fork.

## See also

- [`../tools/subagents.md`](../tools/subagents.md) → "Forks" — full mechanic + comparison table (forks vs named subagents).
- [`../tools/prompt-caching.md`](../tools/prompt-caching.md) — why fork is cache-cheap.
- [`../../shared/workflows/research-then-edit.md`](../../shared/workflows/research-then-edit.md) — when a fresh subagent fits better than a fork.
