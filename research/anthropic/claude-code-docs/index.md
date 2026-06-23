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

**Install methods (cross-OS):**
- **Native installer (recommended; auto-updates in background):**
  - macOS / Linux / WSL: `curl -fsSL https://claude.ai/install.sh | bash`
  - Windows PowerShell: `irm https://claude.ai/install.ps1 | iex`
  - Windows CMD: `curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd`
- **Homebrew (macOS; manual updates):** `brew install --cask claude-code` (stable, ~1 week behind) or `claude-code@latest` (latest channel).
- **WinGet (Windows; manual updates):** `winget install Anthropic.ClaudeCode`.
- **Linux package managers:** apt / dnf / apk (Debian, Fedora, RHEL, Alpine).

**Windows shell note:** Git for Windows is recommended so the Bash tool works natively. Without it, Claude Code uses PowerShell as the shell tool instead. WSL setups don't need Git for Windows.

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
- [MCP (Model Context Protocol)](https://code.claude.com/docs/en/mcp) — open standard for connecting Claude Code to external data sources (Google Drive, Jira, Slack, custom tooling).
- [MCP quickstart](https://code.claude.com/docs/en/mcp-quickstart) — first MCP server end-to-end.
- [Discover plugins](https://code.claude.com/docs/en/discover-plugins) — installing prebuilt plugins from marketplaces.
- [Third-party providers / integrations](https://code.claude.com/docs/en/third-party-integrations) — supported in Terminal CLI and VS Code.

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

### Surfaces

- [Overview](https://code.claude.com/docs/en/features-overview) — platforms & integrations index page.
- [Visual Studio Code](https://code.claude.com/docs/en/vs-code) — install extension; also works with Cursor (`cursor:extension/anthropic.claude-code`).
- [JetBrains IDEs](https://code.claude.com/docs/en/jetbrains) — plugin requires the CLI installed separately.
- [Desktop application](https://code.claude.com/docs/en/desktop) — macOS (Intel/Apple Silicon), Windows x64, Windows ARM64. Visual diff review; receives Dispatch sessions; `/desktop` hand-off from terminal.
- [Desktop quickstart](https://code.claude.com/docs/en/desktop-quickstart).
- [Claude Code on the web](https://code.claude.com/docs/en/claude-code-on-the-web) — `claude.ai/code`; Anthropic-managed cloud sessions; also via iOS app.
- [Web quickstart](https://code.claude.com/docs/en/web-quickstart).
- [Chrome](https://code.claude.com/docs/en/chrome) — debug live web applications from the browser.

### Mobility (move work across devices)

- [Remote Control](https://code.claude.com/docs/en/remote-control) — drive a local session from phone or another device.
- [Channels](https://code.claude.com/docs/en/channels) — push events from Telegram, Discord, iMessage, or custom webhooks into a session.
- **`claude --teleport`** — pull a web/iOS session into the local terminal (requires claude.ai subscription).
- **Dispatch** (covered in [desktop docs](https://code.claude.com/docs/en/desktop)) — message a task from phone; opens a Desktop session.

### CI / automation / scheduled

- [Code Review](https://code.claude.com/docs/en/code-review) — review changes inside Claude Code; pairs with the `/code-review` skill.
- [GitHub Actions](https://code.claude.com/docs/en/github-actions) — `@claude` in PRs/issues to trigger Claude on CI.
- [GitLab CI/CD](https://code.claude.com/docs/en/gitlab-ci-cd) — GitLab pipeline integration.
- [Routines](https://code.claude.com/docs/en/routines) — recurring tasks on Anthropic-managed infra (runs even when your machine is off); triggers from cron, API calls, or GitHub events; created via web, Desktop app, or `/schedule` in CLI.
- [Desktop scheduled tasks](https://code.claude.com/docs/en/desktop-scheduled-tasks) — recurring tasks running on your local machine.
- [`/loop` (scheduled tasks)](https://code.claude.com/docs/en/scheduled-tasks) — repeat a prompt within a CLI session for quick polling.

### Chat & team

- [Slack](https://code.claude.com/docs/en/slack) — mention `@Claude` in Slack; routes to a PR.

### Agent teams

- [Sub-agents](https://code.claude.com/docs/en/sub-agents) — spawn multiple parallel agents with a lead coordinating.
- [Background agents view](https://code.claude.com/docs/en/agent-view) — watch multiple full sessions running in parallel from one screen.

### Other

- **Computer use (preview)** — Claude controls mouse/keyboard/screen. *(URL TBD)*
- **Chrome extension (beta)** — separate from the `/chrome` debug-tool integration. *(URL TBD)*
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
