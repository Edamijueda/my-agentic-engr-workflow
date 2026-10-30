---
verified_against: 2.1.308
verified_on: 2026-10-30
sources:
  - https://code.claude.com/docs/en/plugins/overview
  - https://code.claude.com/docs/en/plugins/security
---

# Plugins

A plugin is a directory of components installed and loaded as one unit. Components can be skills, subagents, hooks, a hooks module (a plugin with one is called a **mod**, and can draw panes and add commands), and MCP servers. A manifest at `.claude-plugin/plugin.json` names it.

## When you need one

**You usually don't.** Skills, subagents, hooks and MCP servers all work standalone — a skill in `~/.claude/skills/` is available in every project already.

A plugin earns its place when you want *several* of those packaged together: to install someone else's setup in one command and get updates, or to hand your own setup to teammates across projects.

## Installing

```bash
/plugin                  # browse; Discover tab lists your marketplaces
claude plugin disable    # stop one without uninstalling
```

Anthropic's official marketplace is added automatically on first interactive session. A marketplace is just a catalog — a repo with `.claude-plugin/marketplace.json` saying what exists and where to fetch it. Install by name: `commit-commands@claude-plugins-official`.

Scopes: **user** (every project on this machine), **project** (everyone in the repo, via committed `.claude/settings.json` — each person still installs it themselves), **local** (this repo, just you).

Cloud sessions don't load the plugins in your local settings.

## What an enabled plugin costs

An enabled plugin is part of **every** session, not only the ones where you use it.

- **Context**: each skill, agent and command Claude can invoke on its own has its name and description in context every turn, so Claude knows it exists. Full bodies load only on use. Bundled MCP servers follow [tool search](mcp.md#the-context-cost-and-why-its-usually-fine).
- **Processes**: its MCP servers run alongside every session where it's enabled; its hooks fire at their events.
- **Permissions**: what it runs, it runs as you.

Check the footprint before installing (official-marketplace plugins show a **Context cost** estimate in `/plugin`) and after (the **Installed** tab groups plugins as **Not used recently**).

## Trust

Name tier is not a safety guarantee — **any** plugin can run code with your privileges. `official` and `community` names are only accepted for marketplaces sourced from `github.com/anthropics/` repos; everything else is third-party, including one your own organisation publishes. Review before installing: [plugin security and trust](https://code.claude.com/docs/en/plugins/security).

## See also

- [`skills.md`](skills.md), [`subagents.md`](subagents.md), [`hooks.md`](hooks.md), [`mcp.md`](mcp.md) — the components, standalone.
- [`../workflows/scaffolding-decision.md`](../workflows/scaffolding-decision.md) — which mechanism fits the friction you found.
- [Create a plugin](https://code.claude.com/docs/en/plugins/create) and [mods](https://code.claude.com/docs/en/plugins/mods/overview) — building your own.
