# my-agentic-engr-workflow

Personal playbook for working efficiently with coding agents. When starting a new machine or a new project, come here first.

## How to use

| Situation | Start here |
| --- | --- |
| Fresh machine | [`shared/project-setup/new-machine.md`](shared/project-setup/new-machine.md) |
| New project | [`shared/project-setup/new-project.md`](shared/project-setup/new-project.md) |
| **You are an agent the user pointed at this playbook from inside another project** | **[`shared/project-setup/setting-up-from-another-project.md`](shared/project-setup/setting-up-from-another-project.md)** |
| Stuck on a task | [`shared/workflows/`](shared/workflows/) — plan-first, research-then-edit, diff-review, long-task-with-compact |
| Spotted recurring friction | [`claude/workflows/scaffolding-decision.md`](claude/workflows/scaffolding-decision.md) |
| Want the "why" behind a choice | [`shared/principles/`](shared/principles/) |
| Specific Claude Code mechanic | [`claude/`](claude/) |

## Layout

- **[`shared/`](shared/)** — agent-agnostic. Should hold regardless of which coding agent you use.
  - **`principles/`** — context-is-finite, enforcement-vs-influence, durability-ladder, defer-until-friction.
  - **`project-setup/`** — new-machine.md (once per machine), new-project.md (once per project).
  - **`workflows/`** — plan-first, research-then-edit, diff-review-cycle, long-task-with-compact.
- **[`claude/`](claude/)** — Claude Code specifics.
  - **`setup/`** — install, authentication, vs-code.
  - **`config/`** — memory, settings, permissions.
  - **`tools/`** — context-window, prompt-caching, hooks, skills, tools-reference, subagents.
  - **`slash-commands/`** — built-in commands + bundled skills reference.
  - **`workflows/`** — parallel-with-forks, custom-subagent-creation, scaffolding-decision, memory-curation.
- **[`research/`](research/)** — scratchpad for raw notes from docs and source material before they get distilled into `shared/` or `claude/`.

## Adding a new agent

If you start using another coding agent (Cursor, Aider, etc.), add a new top-level dir for it (e.g. `cursor/`). Anything cross-cutting between agents gets promoted into `shared/`.

## See also

- [`CLAUDE.md`](CLAUDE.md) — repo context loaded by Claude Code in every session run here.
