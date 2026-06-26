# shared/

Agent-agnostic. Anything here should be true regardless of which coding agent you use. Implementation specifics link out to per-agent dirs (currently [`../claude/`](../claude/)).

## Layout

### [`principles/`](principles/)

Cross-cutting rules that shape every choice in the playbook.

- **[`context-is-finite.md`](principles/context-is-finite.md)** — what you load is what you spend.
- **[`enforcement-vs-influence.md`](principles/enforcement-vs-influence.md)** — memory is influence; permissions and hooks are enforcement. Pick the lightest layer that gives the guarantee.
- **[`durability-ladder.md`](principles/durability-ladder.md)** — rules live at different durability layers; pick the right one for time + machine boundaries.
- **[`defer-until-friction.md`](principles/defer-until-friction.md)** — don't pre-scaffold. Rule of three.

### [`project-setup/`](project-setup/)

Checklists to run when preparing a new context.

- **[`new-machine.md`](project-setup/new-machine.md)** — once per machine. Install, authenticate, drop personal `~/.claude/settings.json`, install VS Code extension, verify. Includes cross-machine sync reality.
- **[`new-project.md`](project-setup/new-project.md)** — once per project. The three things that matter (memory, permissions, plan-mode default), defer-until-friction matrix, pre-flight checklist.

### [`workflows/`](workflows/)

Reusable patterns. Each readable in 60 seconds before starting a task that fits.

- **[`plan-first.md`](workflows/plan-first.md)** — the agent describes before it edits.
- **[`research-then-edit.md`](workflows/research-then-edit.md)** — read-only subagent reads; you edit in main.
- **[`diff-review-cycle.md`](workflows/diff-review-cycle.md)** — the before-you-ship loop.
- **[`long-task-with-compact.md`](workflows/long-task-with-compact.md)** — token-budget management for long sessions.
