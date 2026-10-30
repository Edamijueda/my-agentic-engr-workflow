# CLAUDE.md

Project context for Claude Code (and any other agent that reads this file). Loaded automatically in every session run from this repo.

## What this repo is

One current source of agentic coding practice, so the same guidance isn't retyped in every project. Other projects learn from it; when a project discovers something that works, it gets written back here.

Agent-agnostic on purpose. Claude Code is the agent in use today, but the practice in `shared/` should survive a switch to whatever beats it later — so anything not genuinely Claude-specific belongs there.

## Layout

- `shared/` — agent-agnostic.
  - `principles/` — context-is-finite, enforcement-vs-influence, durability-ladder, defer-until-friction.
  - `project-setup/` — new-machine.md, new-project.md.
  - `workflows/` — plan-first, research-then-edit, diff-review-cycle, long-task-with-compact, doc-freshness.
- `claude/` — Claude Code specifics.
  - `setup/` — install, authentication, vs-code.
  - `config/` — memory, settings, permissions.
  - `tools/` — context-window, prompt-caching, hooks, skills, tools-reference, subagents.
  - `slash-commands/` — built-in commands + bundled skills.
  - `workflows/` — parallel-with-forks, custom-subagent-creation, scaffolding-decision, memory-curation.
- `research/anthropic/` — scratchpad for raw doc notes before they get distilled into `shared/` or `claude/`.

Add a new top-level dir per coding agent as needed (e.g. `cursor/`, `aider/`). Promote cross-cutting findings into `shared/`.

## Conventions

- Drafts and exploration go into `research/`. Once stable, promote into `shared/` or `claude/`.
- **Docs that assert product facts carry a freshness stamp** in YAML frontmatter (`verified_against`, `verified_on`, `sources`). If you change a factual claim in a stamped doc, re-verify it against the live page and update the stamp in the same commit. If you can't verify it, leave the old stamp — it belongs on the stale list. Principles and READMEs are deliberately unstamped. Convention and refresh pass: `shared/workflows/doc-freshness.md`.
- **Keep volatile facts out of `shared/principles/`.** Version numbers, model names and provider names belong in stamped docs; principles should link to them rather than quote them.
- Decide first whether new content is agent-specific or cross-cutting before placing it. If unsure, draft in `research/` and promote later.
- Keep the top-level surface area small — don't add new top-level dirs without checking.
- **Write for macOS**, the platform in use. State an OS-specific fact only where it changes a decision, once, in one line — don't mirror every path and shell snippet across platforms. Link to the official docs for other platforms instead.
- **Scope to the core purpose.** Reference material that duplicates vendor docs doesn't belong here; a few lines plus a link does. Prefer cutting over keeping.
- This repo is **public**. Don't write anything personal or sensitive here; that lives in `~/.claude/` memory on the user's machine.
