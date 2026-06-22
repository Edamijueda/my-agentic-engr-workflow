# Claude Code docs — surface map (breadth pass)

Source: `https://code.claude.com/docs/en/`. Top-level structure mirrors the official sidebar TOC. Captured 2026-06-22.

> WebFetch can't reach `code.claude.com` from this environment (domain block). One-liners are compiled from WebSearch summaries; verify during each deep-dive.
>
> Reference pages (Settings, Hooks, Tools reference, Permissions, Slash commands, Sub-agents, Skills) are real docs pages that aren't all top-level in the sidebar — listed below under the most relevant section.

## Getting started

- [Overview](https://code.claude.com/docs/en/overview) — what Claude Code is, where it runs (terminal, IDE, desktop, web).
- [Quickstart](https://code.claude.com/docs/en/quickstart) — up and running in minutes for common dev tasks.
- [Changelog](https://code.claude.com/docs/en/changelog) — version-by-version changes.
- [What's new](https://code.claude.com/docs/en/whats-new) — weekly digest of notable new features.
- [Advanced setup](https://code.claude.com/docs/en/setup) — install paths beyond quickstart; cross-OS details.
- [Authentication](https://code.claude.com/docs/en/authentication) — Claude (Teams/Enterprise), API console, cloud providers.
- [Troubleshoot install and login](https://code.claude.com/docs/en/troubleshoot-install) — common install/auth issues.
- [Terminal guide for new users](https://code.claude.com/docs/en/terminal-guide) — terminal basics.
- [Glossary](https://code.claude.com/docs/en/glossary) — terminology reference.
- [Docs map](https://code.claude.com/docs/en/claude_code_docs_map) — official site map.

**Cross-OS note:** Windows install via PowerShell (`irm https://claude.ai/install.ps1 | iex`) or CMD (`curl ... install.cmd && install.cmd`). Native Windows works without admin; WSL also supported. Git for Windows recommended so the Bash tool works.

## Core concepts

- [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works) — three phases: gather context, take action, verify.
- [Extend Claude Code](https://code.claude.com/docs/en/features-overview) — overview of the extension surface (sub-agents, skills, hooks, plugins, MCP).
- [Explore the .claude directory](https://code.claude.com/docs/en/claude-directory) — what lives under `.claude/` (project) and `~/.claude/` (global).
- [Explore the context window](https://code.claude.com/docs/en/context) — how context budget works; what eats it and how to manage it. *(URL guess — confirm)*
- [Prompt caching](https://code.claude.com/docs/en/prompt-caching) — caching behavior in Claude Code sessions (token-cost lever). *(URL guess — confirm)*

### Extension surface (nested under Extend Claude Code)

- [Sub-agents](https://code.claude.com/docs/en/sub-agents) — custom subagents with their own context and tool access.
- [Skills](https://code.claude.com/docs/en/skills) — `SKILL.md` files Claude auto-invokes when relevant; stored in `.claude/skills/`.
- [Workflows](https://code.claude.com/docs/en/workflows) — orchestrating subagents at scale.
- [Hooks reference](https://code.claude.com/docs/en/hooks) — events (PreToolUse, Stop, etc.), matchers, handlers.
- [Discover plugins](https://code.claude.com/docs/en/discover-plugins) — installing prebuilt plugins from marketplaces.

## Use Claude Code

- [Store instructions and memories](https://code.claude.com/docs/en/memory) — `CLAUDE.md` for project/user/org context.
- [Permission modes](https://code.claude.com/docs/en/permission-modes) — session-level modes (default, plan, accept-edits, bypass) and when each fits.
- [Manage sessions](https://code.claude.com/docs/en/manage-sessions) — resume, fork, share session state across devices. *(URL guess — confirm)*
- [Common workflows](https://code.claude.com/docs/en/common-workflows) — recipes for frequent tasks.
- [Prompt library](https://code.claude.com/docs/en/prompt-library) — reusable prompts shipped by Anthropic. *(URL guess — confirm)*
- [Best practices](https://code.claude.com/docs/en/best-practices) — environment config, parallel sessions, recurring patterns.

### Reference (nested under Use Claude Code)

- [Settings](https://code.claude.com/docs/en/settings) — `settings.json` for permissions, env vars, tool behavior. Hot-reloaded.
- [Permissions](https://code.claude.com/docs/en/permissions) — allow/deny rules for tools and commands.
- [Commands](https://code.claude.com/docs/en/commands) — overview of in-session controls.
- [Slash commands](https://code.claude.com/docs/en/slash-commands) — built-in (`/plan`, `/mcp`, `/agents`, `/permissions`, `/tasks`, …) plus custom.
- [Tools reference](https://code.claude.com/docs/en/tools-reference) — every built-in tool, parameters, output limits (`BASH_MAX_OUTPUT_LENGTH`, etc.).
- [Sandboxing](https://code.claude.com/docs/en/sandboxing) — sandboxed Bash tool (Seatbelt on macOS, bubblewrap on Linux/WSL2; **note:** native Windows lacks an equivalent).
- [Sandbox environments](https://code.claude.com/docs/en/sandbox-environments) — choosing between sandbox modes.
- [CLI reference](https://code.claude.com/docs/en/cli-reference) — every CLI flag and subcommand.
- [Interactive mode](https://code.claude.com/docs/en/interactive-mode) — keybindings, prompts, in-session behavior.

## Platforms and integrations

- [Overview](https://code.claude.com/docs/en/features-overview) — platforms & integrations index page.
- **Remote Control** — running Claude Code from one device against another. *(URL TBD)*
- [Claude Code on the web](https://code.claude.com/docs/en/claude-code-on-the-web) — browser/phone, Anthropic-managed cloud sessions.
- [Web quickstart](https://code.claude.com/docs/en/web-quickstart) — first cloud session.
- [Desktop application](https://code.claude.com/docs/en/desktop) — Mac/Windows desktop app.
- [Desktop quickstart](https://code.claude.com/docs/en/desktop-quickstart) — first desktop session.
- **Chrome extension (beta)** — Claude Code in the browser. *(URL TBD)*
- **Computer use (preview)** — Claude controls the computer (mouse/keyboard/screen). *(URL TBD)*
- [Visual Studio Code](https://code.claude.com/docs/en/vs-code) — install extension from VS Code marketplace.
- [JetBrains IDEs](https://code.claude.com/docs/en/jetbrains) — install plugin from JetBrains marketplace.
- [Code Review](https://code.claude.com/docs/en/code-review) — review changes inside Claude Code (paired with `/code-review` skill).
- [GitHub Actions](https://code.claude.com/docs/en/github-actions) — `@claude` in PRs/issues to trigger Claude on CI.
- [Routines](https://code.claude.com/docs/en/routines) — scheduled/recurring automated tasks.
- **Claude Code in Slack** — Slack integration. *(URL TBD)*
- [Development containers](https://code.claude.com/docs/en/devcontainer) — running Claude Code in a devcontainer.

## Not in the user-facing sidebar

Surface area I found via search that isn't in the current top-level TOC — likely admin-only sections or moved to sister sites.

### Enterprise / admin

- [Security](https://code.claude.com/docs/en/security)
- [Data usage](https://code.claude.com/docs/en/data-usage)
- [Monitoring](https://code.claude.com/docs/en/monitoring-usage)
- [Identity & access management](https://code.claude.com/docs/en/iam)
- [Team](https://code.claude.com/docs/en/team)
- [Admin setup](https://code.claude.com/docs/en/admin-setup)
- [Enterprise deployment overview](https://code.claude.com/docs/en/bedrock-vertex) — Bedrock / Vertex AI / Microsoft Foundry.
- [Corporate proxy](https://code.claude.com/docs/en/corporate-proxy)
- [GitHub Enterprise Server](https://code.claude.com/docs/en/github-enterprise-server)

### Sister docs site — `platform.claude.com`

Claude API + Agent SDK now live there. Relevant URLs found:

- [Agent SDK overview](https://platform.claude.com/docs/en/agent-sdk/overview)
- [Agent SDK quickstart](https://platform.claude.com/docs/en/agent-sdk/quickstart)
- [Agent SDK — Python](https://platform.claude.com/docs/en/agent-sdk/python) / [TypeScript](https://platform.claude.com/docs/en/agent-sdk/typescript)
- [Use Claude Code features in the SDK](https://platform.claude.com/docs/en/agent-sdk/claude-code-features)
- [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)

A breadth pass on `platform.claude.com` is a follow-up issue — most of that content will feed `shared/principles/` (prompt engineering, agent design) rather than `claude/`.
