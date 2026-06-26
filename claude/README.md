# claude/

Everything specific to Claude Code (CLI, IDE extensions, Agent SDK). Companion to [`../shared/`](../shared/) which holds agent-agnostic content.

## Layout

### [`setup/`](setup/)

Install and authenticate on a new machine.

- **[`install.md`](setup/install.md)** — cross-OS install methods, update channels, version pinning, binary integrity, uninstall.
- **[`authentication.md`](setup/authentication.md)** — login flow, account types, credential storage per OS, 6-tier auth precedence, `setup-token` for CI.
- **[`vs-code.md`](setup/vs-code.md)** — VS Code extension (and Cursor / VS Code forks). The built-in `ide` MCP server and the standalone-CLI-also-needed gotcha.

### [`config/`](config/)

The three foundational config layers.

- **[`memory.md`](config/memory.md)** — `CLAUDE.md` scope ladder, `.claude/rules/`, imports, `/init`, `/compact` behavior.
- **[`settings.md`](config/settings.md)** — `settings.json` scopes + precedence, hot-reload, the curated useful keys.
- **[`permissions.md`](config/permissions.md)** — allow/ask/deny rules, all six permission modes, tool-specific patterns.

### [`tools/`](tools/)

How Claude Code's engine works under the hood.

- **[`context-window.md`](tools/context-window.md)** — what fills the 200K budget, what survives `/compact`.
- **[`prompt-caching.md`](tools/prompt-caching.md)** — what invalidates the cache vs. what keeps it (incl. the per-machine + per-directory scope rule).
- **[`hooks.md`](tools/hooks.md)** — 28 lifecycle events, 5 handler types, JSON/exit-code output protocol.
- **[`skills.md`](tools/skills.md)** — `SKILL.md` anatomy, frontmatter, dynamic context injection, invocation control, lifecycle.
- **[`subagents.md`](tools/subagents.md)** — built-in + custom subagents, scope ladder, frontmatter, foreground/background, forks.
- **[`tools-reference.md`](tools/tools-reference.md)** — canonical names of all built-in tools, per-tool behavior, rule formats.

### [`slash-commands/`](slash-commands/)

- **[`built-in.md`](slash-commands/built-in.md)** — built-in commands + bundled skills, organized by workflow phase.

> Custom slash commands are skills now — see [`tools/skills.md`](tools/skills.md).

### [`workflows/`](workflows/)

Claude-specific patterns. Companion to [`../shared/workflows/`](../shared/workflows/) (agent-agnostic).

- **[`scaffolding-decision.md`](workflows/scaffolding-decision.md)** — you spotted friction; which mechanism do you reach for?
- **[`memory-curation.md`](workflows/memory-curation.md)** — maintaining `CLAUDE.md` over time.
- **[`custom-subagent-creation.md`](workflows/custom-subagent-creation.md)** — graduating from built-in `Explore`.
- **[`parallel-with-forks.md`](workflows/parallel-with-forks.md)** — `/fork` for cheap A/B exploration.
