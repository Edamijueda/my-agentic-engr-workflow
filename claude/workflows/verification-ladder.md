---
verified_against: 2.1.308
verified_on: 2026-10-30
sources:
  - https://code.claude.com/docs/en/goal
  - https://code.claude.com/docs/en/hooks
  - https://code.claude.com/docs/en/best-practices
---

# Verification ladder (Claude Code)

Which mechanism implements each rung of [`checks-over-claims.md`](../../shared/principles/checks-over-claims.md). Climb only as far as the work needs.

## Rung 1 — say it in the prompt

```
run the auth tests after, and keep going until they pass
```

No setup, works today, covers most tasks. Claude decides when it's satisfied, which is fine when you're watching.

Then ask for the output, not the verdict.

## Rung 2 — `/goal`

```
/goal all tests in test/auth pass and the lint step is clean
```

A separate evaluator — your configured small/fast model — reads the conversation after **every** turn and returns one of three verdicts: *not yet met* (Claude keeps working and takes the reason as guidance), *met* (goal clears, recorded as achieved), or *impossible* (goal clears, recorded as failed). Setting a goal starts a turn immediately; the condition is the directive.

**The constraint that catches people out: the evaluator cannot run commands or read files.** It only judges what Claude has already surfaced in the conversation. So write conditions that Claude's own output can demonstrate — "all tests in `test/auth` pass" works because Claude runs them and the result lands in the transcript. A condition about state nobody printed can never be judged met.

A condition that holds up over many turns has one measurable end state, a stated way to prove it (`npm test` exits 0, `git status` is clean), and any constraint that must not be violated on the way. Up to 4,000 characters. Add `or stop after 20 turns` to bound the run.

Worth knowing:

- One goal per session. `/goal` alone shows status (condition, duration, turns evaluated, token spend, latest reason); `/goal clear` removes it.
- **It doesn't change your permission mode.** For unattended turns, pair it with auto mode — otherwise Claude still stops to ask about anything your settings don't already allow.
- If Claude keeps answering the evaluator without touching a tool, Claude Code breaks the loop and hands control back with the goal still set.
- Background work defers evaluation until a turn ends with nothing running.
- Survives resume: the condition carries over, the counters reset.
- Works under `-p`, which runs the whole loop in one invocation. Add `--output-format stream-json --verbose` or it looks hung.
- Unavailable when hooks are off (`disableAllHooks`, `allowManagedHooksOnly`), because it *is* a hook — see below.

## Rung 3 — a `Stop` hook

`/goal` is a session-scoped, prompt-based `Stop` hook with the condition filled in. Write the hook yourself when the check should be deterministic and apply to every session in scope rather than the one you typed it into.

```json
{
  "hooks": {
    "Stop": [
      { "hooks": [{ "type": "command", "command": "pnpm tsc --noEmit 1>&2 || exit 2" }] }
    ]
  }
}
```

**Exit code 2 is what blocks the stop**, and whatever the command writes to **stderr** goes back to Claude as the reason it can't finish — which is why the compiler output is redirected there. Claude never sees stderr from a hook that exits 0.

The cap: if the hook blocks **eight stops in a row** with no tool call in between, Claude Code overrides the next block and ends the turn anyway. Raise it with `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP`. Without that cap a failing check would trap the session.

Full mechanics, including `additionalContext` vs `decision: "block"`: [`../tools/hooks.md`](../tools/hooks.md).

## Rung 4 — a fresh reviewer

`/code-review` runs in a subagent that never saw your conversation, so it isn't persuaded by the reasoning that produced the code. `/verify` goes further and has Claude build, run and look at the app — catching what tests and typechecks miss.

See [`../../shared/workflows/diff-review-cycle.md`](../../shared/workflows/diff-review-cycle.md).

## Choosing

| Situation | Rung |
| --- | --- |
| You're at the keyboard | 1 |
| Long run you're not watching | 2 |
| A check that must pass in every session, enforced | 3 |
| About to ship something you'd not bet on | 4 |

Rungs combine: a goal for the loop, a stop hook for the floor, a review before the PR.

## See also

- [`../../shared/principles/checks-over-claims.md`](../../shared/principles/checks-over-claims.md) — why any of this beats trusting a summary.
- [`../tools/hooks.md`](../tools/hooks.md) — the hooks reference.
- [`../slash-commands/built-in.md`](../slash-commands/built-in.md) — `/goal`, `/verify`, `/code-review` in the command reference.
