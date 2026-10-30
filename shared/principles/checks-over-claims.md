# Checks over claims

An agent stops when the work *looks* done. Give it something that comes back pass or fail and it can tell the difference itself. Without that, you are the test suite.

## Why it's true

"Looks done" is all an agent has to go on unless something external disagrees with it. That has two consequences, and the second is the expensive one:

1. **Mistakes survive until you happen to notice.** Every error sits there, in code you didn't write, until you read it.
2. **The agent can't iterate.** Given a check, it makes the change, runs it, reads the failure, and fixes it — a loop that needs no input from you. Without one, it hands you work of unknown quality and waits.

The check doesn't have to be a test suite. Anything the agent can run and read the result of will do: a build's exit code, a linter, a typecheck, a script that diffs output against a fixture, a rendered page or screenshot next to the mockup (see [`form-follows-use.md`](form-follows-use.md)). If your project already has one, the single most useful line in your instruction file is the command that runs it.

## Ask for evidence, not a verdict

"All tests pass" tells you almost nothing. Ask for the command and its output.

This is cheap to demand and it changes what you're reviewing: reading the output takes less time than re-running the check yourself, and for a session you weren't watching it's the only evidence that exists. An agent that says it's done and shows you nothing has told you what it believes, not what happened.

## The ladder

Four rungs, loosest to strictest. Climb only as far as the work needs.

| Rung | What decides "done" | When it's worth it |
| --- | --- | --- |
| **In the prompt** — "run the tests after, and keep going until they pass" | The agent | Any task, right now, no setup. Covers most work. |
| **A goal condition** | A separate evaluator, after every turn | Long unattended runs, where the agent might end a turn with a progress summary while items are still open. |
| **A stop hook** | Your script — the harness enforces it | A check that must pass before any turn can end, in every session. |
| **A fresh reviewer** | Something that never saw your conversation | Changes you wouldn't bet money on. See [`../workflows/diff-review-cycle.md`](../workflows/diff-review-cycle.md). |

The first rung is doing the work in most cases. Reach past it when a specific failure keeps happening, not pre-emptively — see [`defer-until-friction.md`](defer-until-friction.md).

Rungs two and three are the same move as [`enforcement-vs-influence.md`](enforcement-vs-influence.md): the prompt is influence, the hook is enforcement. A goal condition sits between them — evaluated by a model, but one that isn't the model doing the work.

## How to apply

- **Put the check command in the instruction file.** Not the architecture, not the layout — the command. It's the line that earns its place most reliably.
- **Say what counts as done when you assign the work.** "Keep going until `npm test` exits 0" is a complete specification of done; "fix the auth bug" isn't.
- **Make the check cheap to run.** A check that takes ten minutes won't get run in a loop. Scope it to what the change touches.
- **Don't accept a summary as proof.** Ask what command ran and what it printed.
- **Review against the spec, not for bugs in general.** A reviewer asked to find problems will always find something; fix every finding and you get abstraction layers and tests for impossible cases. Name what to skip.

## See also

- [`../workflows/diff-review-cycle.md`](../workflows/diff-review-cycle.md) — the before-you-ship loop, including reviewing against a spec.
- [`../../claude/workflows/verification-ladder.md`](../../claude/workflows/verification-ladder.md) — which Claude Code mechanism implements each rung.
- [`enforcement-vs-influence.md`](enforcement-vs-influence.md) — why the hook rung is categorically different from the prompt rung.
- [Best practices: give Claude a check to run](https://code.claude.com/docs/en/best-practices) — the source for this principle.
