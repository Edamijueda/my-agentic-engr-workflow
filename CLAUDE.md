# CLAUDE.md

Project context for Claude Code (and any other agent that reads this file). Loaded automatically in every session run from this repo.

## What this repo is

A personal playbook for working efficiently with coding agents. The user comes here when starting a new project, to prepare both the machine and the project for agent-friendly work — install steps, configs, principles, reusable workflows.

## Layout

- `shared/` — agent-agnostic.
  - `principles/` — context-is-finite, enforcement-vs-influence, durability-ladder, defer-until-friction.
  - `project-setup/` — new-machine.md, new-project.md.
  - `workflows/` — plan-first, research-then-edit, diff-review-cycle, long-task-with-compact.
- `claude/` — Claude Code specifics.
  - `setup/` — install, authentication, vs-code.
  - `config/` — memory, settings, permissions.
  - `tools/` — context-window, prompt-caching, hooks, skills, tools-reference, subagents.
  - `slash-commands/` — built-in commands + bundled skills.
  - `workflows/` — parallel-with-forks, custom-subagent-creation, scaffolding-decision, memory-curation.
- `research/anthropic/` — scratchpad for raw doc notes before they get distilled into `shared/` or `claude/`.

Add a new top-level dir per coding agent as needed (e.g. `cursor/`, `aider/`). Promote cross-cutting findings into `shared/`.

## Cross-OS scope

The playbook must work on both **macOS** and **Windows**. User is on macOS most of the time but switches to Windows (running in VMware Fusion) for some client projects.

When writing setup steps, install instructions, paths, or shell snippets:
- Cover both OSes — either side-by-side or under clear OS headings.
- Prefer OS-neutral phrasing where it doesn't cost clarity.
- If a feature genuinely only works on one OS, flag it.

## Conventions

- Drafts and exploration go into `research/`. Once stable, promote into `shared/` or `claude/`.
- Decide first whether new content is agent-specific or cross-cutting before placing it. If unsure, draft in `research/` and promote later.
- Keep the top-level surface area small — don't add new top-level dirs without checking.
- This repo is **public**. Don't write anything personal or sensitive here; that lives in `~/.claude/` memory on the user's machine.
