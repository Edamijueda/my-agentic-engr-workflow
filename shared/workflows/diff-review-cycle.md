# Diff-review cycle

The before-you-ship loop. Use `/diff` / `/code-review` / `/simplify` / `/rewind` together to catch issues without re-thinking the work yourself.

## When to use it

- **Before committing** a non-trivial change.
- **After a long task session** before you push.
- **Before opening a PR** — the same engine runs server-side via `/code-review ultra` for deep cloud review.
- **When something feels off** but you can't articulate what.

If the change is two lines of obvious cleanup, skip it. The cycle is for changes you wouldn't bet money on.

## The tools

| Tool | What it does |
| --- | --- |
| **`/diff`** | Interactive side-by-side viewer. You read what changed. |
| **`/code-review`** | Reviews the current diff for **correctness bugs + reuse/simplify/efficiency cleanups**. Effort levels: `low`/`medium`/`high`/`xhigh`/`max`/`ultra`. Optional `--fix` applies findings; `--comment` posts as inline PR comments. |
| **`/simplify`** | **Cleanup-only review** (v2.1.154+). Four parallel agents: reuse, simplification, efficiency, abstraction level. No bug hunting — use `/code-review` for that. |
| **`/security-review`** | Focused security pass. Injection, auth, data exposure. |
| **`/rewind`** | Roll the conversation + code back to an earlier turn. Cheaper than `/compact` — preserves prompt cache. |

## The standard loop

```
/diff                          # 1. read what changed yourself
/code-review                   # 2. automated bug + cleanup pass
                              # 3. agree → /code-review --fix or fix manually
                              #    disagree → explain why, continue
/simplify                      # 4. (optional) cleanup-only pass
                              # 5. commit when satisfied
```

### Step-by-step

1. **`/diff` first.** Read the changes yourself. This catches the dumb mistakes — extra parens, accidentally-edited unrelated file, debugging `console.log` left behind. Don't skip this trusting the agent.
2. **`/code-review`** (or `/code-review high` for deeper review, or `/code-review ultra` for the multi-agent cloud version on big changes). Reads as suggestions — not ground truth.
3. **Triage findings.**
   - **Agree**: `/code-review --fix` to apply, or describe the fix to Claude and let it apply.
   - **Disagree**: tell Claude why the finding doesn't apply, move on. The review engine isn't perfect.
4. **`/simplify`** (optional). If the change involves a lot of new code, this pass catches duplication and over-abstraction.
5. **Commit** when satisfied.

## When a path turned out wrong

If review reveals the change went the wrong direction (not just bugs but wrong approach):

```
/rewind
```

Pick the message to roll back to. The conversation truncates, the code reverts to that checkpoint. **You re-explain from there, not from scratch** — the conversation up to that point is preserved.

`/rewind` is cheaper than `/compact` because the cache still matches (the prefix hasn't changed). See [`claude/tools/prompt-caching.md`](../../claude/tools/prompt-caching.md).

`/rewind` offers three options:
- **Fork from here**: new branch, keep code changes intact.
- **Rewind code to here**: revert files, keep full conversation history.
- **Fork and rewind code**: both.

For "I went down a wrong path entirely" → option 2 or 3.

## Variations

### Deep PR review

`/code-review ultra` (alias `/ultrareview`) — multi-agent cloud review. Slower (minutes, not seconds), more thorough. Worth it for:
- PRs touching critical code (auth, payments, prod config).
- Big refactors.
- Anything you'd want a senior to review before merge.

3 free runs/week on Pro and Max; usage credits beyond that.

### Reviewing a teammate's PR

```
/review <PR-URL-or-number>
```

Same engine, but applied to a GitHub PR you check out or specify. Combine with `--comment` to post findings as inline review comments on GitHub.

### Security-focused pass

```
/security-review
```

For changes touching auth, input handling, file uploads, deserialization, or anything that takes user input.

## Common mistakes

- **Skipping `/diff` because you trust the agent.** Pure laziness. The first read catches the silly mistakes that automated review will miss (forgotten debug logs, comments not removed, unrelated changes).
- **Treating `/code-review` as ground truth.** Read each finding. Agree or disagree on the merits, not the source.
- **Using `/compact` to abandon a wrong path.** `/rewind` does the same thing better — preserves cache, preserves history up to the point you want to revert to.
- **Running `/code-review` after every micro-edit.** Burns tokens on negligible deltas. Run it on coherent units of change (one feature, one bug fix).
- **`/code-review --fix` without reading the findings first.** You should approve the *direction* of fixes before applying them automatically.

## Per-tool implementation

For Claude Code:
- `/code-review [low|medium|high|xhigh|max|ultra] [--fix] [--comment] [target]` — full effort/fix/comment matrix.
- `/code-review ultra` and `/ultrareview` are aliases for cloud review.
- `/diff` is interactive — left/right arrows switch between current git diff and individual Claude turns.
- `/rewind` aliases: `/checkpoint`, `/undo`.

Full command reference: [`claude/slash-commands/built-in.md`](../../claude/slash-commands/built-in.md) → "Before you ship".

## See also

- [`plan-first.md`](plan-first.md) — plan-then-execute is the upstream half of "agent built it right"; this workflow is the downstream half of "ship it right."
- [`long-task-with-compact.md`](long-task-with-compact.md) — when `/rewind` is the right escape vs. `/compact` vs. `/clear`.
- [`../principles/enforcement-vs-influence.md`](../principles/enforcement-vs-influence.md) — review is influence (suggestions, you decide); permissions are enforcement.
