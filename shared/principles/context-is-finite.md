# Context is finite

What you load is what you spend — in money, in latency, and in the agent's attention. Keeping context focused is not about avoiding a cliff. It's about getting better answers for less.

## What changed

This principle used to say: the context window is your main constraint, so manage it aggressively. That was true when windows were small and compaction was something you had to stay ahead of.

It isn't the binding constraint any more. Windows are large, and compaction runs on its own before you hit the limit. Running out is now rare enough that designing around it is wasted effort — and the workarounds people built for it (delegating reads purely to save tokens, pre-emptive compaction) cost more than they save.

What survived is the part that was never really about the limit:

- **Cost.** Every token in the prefix is re-sent on every turn. Cached, it's cheap but not free.
- **Attention.** A short instruction file gets followed more reliably than a long one. This is the effect that matters most and the one people notice least.
- **Focus.** An agent carrying three abandoned approaches gives worse answers than one carrying the current task.

So: still worth managing, for different reasons than before.

## How to apply

- **Be specific.** "Fix the bug in `auth.ts` line 42" beats "fix the auth bug" — not because reads are expensive, but because a vague prompt makes the agent guess, and guessing is what produces the wrong change.
- **Keep instruction files short.** Under a couple of hundred lines. Past that, adherence drops. If a file is getting long, split it so each part loads only when relevant rather than making it louder.
- **Don't describe what the agent can read.** Directory layouts, dependency lists, architecture overviews — it reads the repo. Write down only what isn't in there: the deploy step that lives in someone's head, the constraint with a reason, the convention that differs from the tool's default.
- **`/clear` between unrelated tasks.** The cheapest lever there is. A focused context answers better and costs less; old conversation crowds out what you need next.
- **Hide side-effect commands from auto-discovery** so they cost nothing until you invoke them by name.
- **Ask for the answer, not the material.** "List the three files with the most TODOs" beats "show me all TODOs."

## What not to do

- **Don't delegate to a subagent to save context.** A subagent is a *second* conversation with its own system prompt and its own tokens — usually more total spend, not less. Delegate for the reasons that actually hold: independent work in parallel, keeping noisy output out of your way, or review by something that hasn't seen your reasoning. See [`../workflows/research-then-edit.md`](../workflows/research-then-edit.md).
- **Don't add emphasis to make a rule stick.** Capitals and IMPORTANT markers make the agent over-apply the rule, and a line that keeps getting missed in a long file means the file is too long. If it genuinely cannot bend, it isn't an instruction — see [`enforcement-vs-influence.md`](enforcement-vs-influence.md).
- **Don't compact pre-emptively.** Compaction has a cost and discards detail. Do it at a natural break, or rewind instead if you're abandoning a path.

## See also

- [`claude/tools/context-window.md`](../../claude/tools/context-window.md) — what fills the budget and what survives compaction.
- [`claude/tools/prompt-caching.md`](../../claude/tools/prompt-caching.md) — what a cached prefix costs and what invalidates it.
- [`../workflows/long-task-with-compact.md`](../workflows/long-task-with-compact.md) — the levers, and when each is the right one.
- [Best practices: what you can stop doing](https://code.claude.com/docs/en/best-practices) — the source for the retirement above.
