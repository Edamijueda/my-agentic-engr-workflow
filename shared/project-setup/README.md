# project-setup/

Checklists for preparing a project to work well with coding agents. Agent-agnostic at the top level — implementation specifics link out to `../../claude/` (or eventually `../../cursor/`, etc.).

## Current docs

- **[`new-machine.md`](new-machine.md)** — once per machine. Install, authenticate, drop personal `~/.claude/settings.json`, install VS Code extension, verify. Includes cross-machine sync reality (what crosses Mac ↔ Windows VM and what doesn't).
- **[`new-project.md`](new-project.md)** — once per project. The three things that matter (memory, permissions, plan-mode default), optional scaffolding to defer until friction shows up, pre-flight checklist with literal commands in order.

Run order: `new-machine.md` first (once per machine), then `new-project.md` per project.

## Deferred

- **`mid-project-handoff.md`** — what to do when joining a project Claude Code has already been used on (other developers' settings to honor, `.claude/` dir audit, memory file curation).

## Why this lives in `shared/`

These are checklists where the *steps* are agent-agnostic (write a memory file, set permission rules, default to plan mode), even though the *implementation* is Claude-specific. Someone using Cursor or Aider can follow the structure and substitute their agent's equivalents. When a step is irreducibly Claude-specific (e.g. `/init` command), the doc links out to `claude/` rather than embedding.
