---
verified_against: 2.1.308
verified_on: 2026-10-30
sources:
  - https://code.claude.com/docs/en/sub-agents
  - https://code.claude.com/docs/en/best-practices
---

# Research-then-edit

A read-only subagent researches; you edit in main with its summary as focused context.

## Why this works

**Not to save tokens.** This doc used to claim ~15× savings on a typical research task. That framing is retired: a subagent is a second conversation with its own system prompt and its own billing, so delegating usually costs *more* in total than reading the files yourself. Handing the same task to a subagent also doesn't make the answer better.

What it actually buys you:

- **Noise stays out of your way.** A wide search produces a lot of output you don't need to see. The subagent absorbs it and hands back the conclusion.
- **Parallelism.** Several independent questions can be answered at once, which reading sequentially can't do.
- **A reader without your assumptions.** A subagent that never saw your conversation isn't anchored on the explanation you've already half-committed to. This is the one case where delegation genuinely improves the answer — and it's why independent review works.

Pick it for those reasons. If none of them apply, just read the files.

## When to use it

- **Wide, noisy searches** — "find all callers of X", "audit every usage of Y". Bounded question, clear deliverable, output you don't want in full.
- **Several independent questions at once** — see Parallel research below.
- **Review that shouldn't inherit your reasoning** — see [`diff-review-cycle.md`](diff-review-cycle.md).
- **"Trace this bug back to its source"** — exploratory work across many files where you want the conclusion, not the trail.

## When NOT to use it

- **Single-file question.** Just open the file or ask in main.
- **Quick check** (does this function exist? what's the signature?). Subagents have startup overhead and cost extra.
- **Anywhere you'd only be doing it "for context".** That reason no longer holds — see [`../principles/context-is-finite.md`](../principles/context-is-finite.md).
- **Iterative refinement** where you'll need to follow up several times — subagents start fresh each time and need re-briefing.
- **Tasks that require both reading AND editing in one pass.** Defeats the isolation.

## The pattern

1. **Phrase the research with the output you want.** "Find all places that call `auth.refreshToken` and return a summary of how each handles errors" beats "look at the auth module."
2. **Specify the subagent.** "Use the Explore subagent" for read-only research (Haiku-fast, cheap). Use `general-purpose` if the task might need writes. For Claude Code: see [`claude/tools/subagents.md`](../../claude/tools/subagents.md) for the built-in roster.
3. **Wait for the summary.** Main session shows a brief notice that a subagent is working; the summary returns when it finishes.
4. **Make the change in main**, with the summary as focused context. You have the synthesized answer, not the raw file contents.

### Example invocation

```
Use the Explore subagent to find every place in this repo that constructs
an HTTP response with a 4xx status, and summarize the error-message
patterns being used. I want to standardize them.
```

Then in main:

```
Based on the subagent's summary, standardize all 4xx responses to use
the format described in option 2.
```

## Variations

### Parallel research

When you have 3+ independent areas to research, spawn them concurrently:

```
Research the auth, database, and API modules in parallel using separate subagents.
Each subagent should return a 1-paragraph summary of its module's role and key files.
```

Best when the research paths don't depend on each other. Claude synthesizes at the end.

> **Each summary still lands in main**, and each subagent bills separately. Parallelism is the gain here; economy isn't. For sustained parallel work, agent teams are the heavier tool (separate doc; deferred).

### Chain subagents

```
Use the code-reviewer subagent to find performance issues, then use the
optimizer subagent to fix them.
```

Each subagent gets the previous one's summary as context. Useful when a multi-step pipeline doesn't fit one subagent's bounded scope.

### Custom research subagent (after the rule of three)

If you're delegating the same kind of research repeatedly, define a custom subagent in `.claude/agents/` with `memory: project` (see [`claude/tools/subagents.md`](../../claude/tools/subagents.md)). It accumulates project conventions across sessions. Don't write this on day one — wait for the third repeat (see [`../principles/defer-until-friction.md`](../principles/defer-until-friction.md)).

## Common mistakes

- **Asking the subagent to BOTH research AND edit.** Defeats the isolation — the edit gets the raw file content via the subagent's reads, but the changes only land if the subagent itself writes (and then you've lost main-session control of the edit). Keep research and editing separate.
- **Vague delegation: "look at the auth module"** → unbounded reads → big summary back. Specify the deliverable.
- **Over-spawning.** 10 parallel subagents each returning a detailed summary can crowd main worse than reading the files yourself would have. ~3-5 parallel is a reasonable upper bound for typical work.
- **Using a subagent for a quick check.** Subagent startup loads CLAUDE.md, env, and the task prompt — overhead for a one-file question.

## Per-tool implementation

For Claude Code:
- **`Explore`** built-in subagent — Haiku, read-only (no Write/Edit). Specify thoroughness: `quick` / `medium` / `very thorough`.
- **`Plan`** built-in subagent — used during plan mode, read-only, inherits model.
- **`general-purpose`** built-in subagent — inherits model, all tools. Use when research might lead to writes.
- **Explore and Plan skip CLAUDE.md and git status** — keeps research fast and cheap, but means project conventions don't reach the subagent. Restate critical rules in the delegation prompt.

Full mechanics: [`claude/tools/subagents.md`](../../claude/tools/subagents.md).

## See also

- [`plan-first.md`](plan-first.md) — pairs naturally. Plan mode for the plan; research-then-edit during execution if the plan needs deep exploration.
- [`long-task-with-compact.md`](long-task-with-compact.md) — the levers for keeping a long session focused.
- [`../principles/context-is-finite.md`](../principles/context-is-finite.md) — why "to save context" is no longer a reason to delegate.
- [`diff-review-cycle.md`](diff-review-cycle.md) — the fresh-reviewer case, which is the strongest reason to delegate.
- [`../principles/defer-until-friction.md`](../principles/defer-until-friction.md) — when to write a custom subagent vs. using built-ins.
