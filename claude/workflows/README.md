# workflows/

Patterns specific to Claude Code — things that leverage mechanics other agents don't have. Companion to [`../../shared/workflows/`](../../shared/workflows/) (agent-agnostic patterns).

Goal: each doc readable in 60-90 seconds before starting a task that fits the pattern.

## Current docs

- **[`parallel-with-forks.md`](parallel-with-forks.md)** — `/fork` for multiple approaches from the same starting point. Cache-shared with main = cheap. Panel controls, worktree isolation, common mistakes.
- **[`custom-subagent-creation.md`](custom-subagent-creation.md)** — graduating from built-in `Explore`. Rule-of-three trigger, frontmatter starter, `description` that drives delegation, `memory: project` for cross-session learning.
- **[`scaffolding-decision.md`](scaffolding-decision.md)** — you spotted friction worth scaffolding. Which mechanism? Decision tree across memory / skills / permissions / hooks / subagents / MCP. Cost comparison + worked examples.
- **[`memory-curation.md`](memory-curation.md)** — maintaining `CLAUDE.md` over time. Splitting strategies (`.claude/rules/` with `paths:` vs `@imports` vs skills), `/init` regenerate, auto-memory complement, the path-scoped-rule `/compact` gotcha.

## Reading order

If you're new to Claude Code workflows beyond the basics:

1. **scaffolding-decision** — orients you on which mechanism fits which friction.
2. **memory-curation** — the most-touched scaffolding (CLAUDE.md grows on every project).
3. **custom-subagent-creation** — when memory + skills aren't enough.
4. **parallel-with-forks** — situational; reach for it when you need it.

The "decision before doing" docs (scaffolding-decision, memory-curation) inform the "create the thing" docs (custom-subagent-creation, parallel-with-forks).
