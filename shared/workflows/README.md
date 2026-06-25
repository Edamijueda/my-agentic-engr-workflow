# workflows/

Reusable patterns — what to do when you find yourself in a familiar situation. Agent-agnostic at the top level; Claude-specific commands link to `../../claude/`.

Goal: each doc readable in 60 seconds before starting a task that fits the pattern.

## Current docs

- **[`plan-first.md`](plan-first.md)** — the agent describes before it edits. When plan mode earns its overhead; accept-into-which-mode decision (auto vs acceptEdits vs default).
- **[`research-then-edit.md`](research-then-edit.md)** — read-only subagent reads, you edit in main. The cheapest token-savings pattern (~15× in typical research).
- **[`diff-review-cycle.md`](diff-review-cycle.md)** — `/diff` / `/code-review` / `/simplify` / `/rewind` together. The before-you-ship loop.
- **[`long-task-with-compact.md`](long-task-with-compact.md)** — `/context` / `/compact` / `/clear` / `/rewind` decision tree. Token-budget management for sessions over an hour.

## Reading order

If you've never used these workflows:

1. **plan-first** — foundational. Most other workflows assume you're in plan-then-execute rhythm.
2. **research-then-edit** — the lever you'll reach for most often.
3. **long-task-with-compact** — for when you stay in a session long enough to need it.
4. **diff-review-cycle** — the closing half before commit/PR.
