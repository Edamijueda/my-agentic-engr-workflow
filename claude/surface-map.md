---
verified_against: 2.1.308
verified_on: 2026-10-30
sources:
  - https://code.claude.com/docs/llms.txt
---

# The rest of the surface

Claude Code is much larger than what this playbook documents in depth. That's deliberate: a doc here has to earn its maintenance cost, and most of this surface is either niche, or well enough covered by the official page that copying it would just create something to go stale.

This is the index. One line each, with a link. Reach for one when you actually need it.

**Documented in depth here instead:** [memory](config/memory.md) · [settings](config/settings.md) · [permissions](config/permissions.md) · [context window](tools/context-window.md) · [prompt caching](tools/prompt-caching.md) · [hooks](tools/hooks.md) · [skills](tools/skills.md) · [subagents](tools/subagents.md) · [tools reference](tools/tools-reference.md) · [MCP](tools/mcp.md) · [plugins](tools/plugins.md) · [slash commands](slash-commands/built-in.md) · [install](setup/install.md) · [auth](setup/authentication.md) · [VS Code](setup/vs-code.md)

## Running work you aren't watching

- [Headless / programmatic](https://code.claude.com/docs/en/headless) — `claude -p`, output formats, `--max-turns`, `--max-budget-usd`. The entry point for scripting Claude Code.
- [Scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks) — `/loop` to repeat a prompt in-session, and the comparison of scheduling options.
- [Routines](https://code.claude.com/docs/en/routines) — recurring work on Anthropic's infra, running when your machine is off. `/schedule` in the CLI.
- [Keep Claude working toward a goal](https://code.claude.com/docs/en/goal) — `/goal`. Covered as rung 2 of [the verification ladder](workflows/verification-ladder.md).
- [GitHub Actions](https://code.claude.com/docs/en/github-actions) — `@claude` in PRs and issues. [GitLab CI/CD](https://code.claude.com/docs/en/gitlab-ci-cd) for GitLab pipelines.

## Many agents at once

- [Run agents in parallel](https://code.claude.com/docs/en/agents) — the overview of the parallelism options.
- [Agent teams](https://code.claude.com/docs/en/agent-teams) — a lead coordinating several sessions, with [cross-session messaging](https://code.claude.com/docs/en/cross-session-messaging).
- [Dynamic workflows](https://code.claude.com/docs/en/workflows) — orchestrating subagents at scale, including fan-out.
- [Agent view](https://code.claude.com/docs/en/agent-view) — watching parallel sessions from one screen.
- [Worktrees](https://code.claude.com/docs/en/worktrees) — parallel sessions on isolated checkouts. Partly covered in [`workflows/parallel-with-forks.md`](workflows/parallel-with-forks.md).

## Other surfaces

- [Claude Code on the web](https://code.claude.com/docs/en/claude-code-on-the-web) — cloud sessions at `claude.ai/code`.
- [Desktop app](https://code.claude.com/docs/en/desktop) — visual diff review; `/desktop` hands a session over.
- [Remote Control](https://code.claude.com/docs/en/remote-control) — drive a local session from another device. `/teleport` pulls a cloud session into your terminal.
- [Chrome](https://code.claude.com/docs/en/chrome) — debugging a live web app from the browser.
- [Slack](https://code.claude.com/docs/en/slack) · [Channels](https://code.claude.com/docs/en/channels) — push events into a running session from Telegram, Discord, iMessage or a webhook.
- [Computer use](https://code.claude.com/docs/en/computer-use) — mouse, keyboard and screen control from the CLI.

## Review and security

- [Code Review](https://code.claude.com/docs/en/code-review) — the hosted product, beyond the `/code-review` skill. [Local review](https://code.claude.com/docs/en/code-review-local) and [ultrareview](https://code.claude.com/docs/en/ultrareview) cover the in-session paths; see [`../shared/workflows/diff-review-cycle.md`](../shared/workflows/diff-review-cycle.md).
- [Security guidance](https://code.claude.com/docs/en/security-guidance) — catching issues as code is written. [Scan a codebase](https://code.claude.com/docs/en/claude-security).
- [Sandboxing](https://code.claude.com/docs/en/sandboxing) — sandboxed Bash (Seatbelt on macOS). Summarised in [`config/settings.md`](config/settings.md).

## Artifacts

- [Share session output as artifacts](https://code.claude.com/docs/en/artifacts) — publishing a page from a session. Reach for it when output is meant for someone else to look at or use rather than read as text. **Not yet distilled here** — see the next-gaps list in [`../research/anthropic/claude-code-docs/priorities.md`](../research/anthropic/claude-code-docs/priorities.md).

## Scaling up and debugging

- [Large codebases and monorepos](https://code.claude.com/docs/en/large-codebases) — setup for repos where the defaults strain.
- [Troubleshooting](https://code.claude.com/docs/en/troubleshooting) · [Debug your config](https://code.claude.com/docs/en/debug-your-config) · [Error reference](https://code.claude.com/docs/en/errors).
- [CLI reference](https://code.claude.com/docs/en/cli-reference) · [Interactive mode](https://code.claude.com/docs/en/interactive-mode) · [Sessions](https://code.claude.com/docs/en/sessions) — reference material; skim, don't distil.
- [Devcontainers](https://code.claude.com/docs/en/devcontainer) — if a project ships one.

## Reference you'll want eventually

- [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works) — the loop, and what loads at startup.
- [Extend Claude Code](https://code.claude.com/docs/en/features-overview) — the official framing of skills vs hooks vs MCP vs subagents vs plugins.
- [Best practices](https://code.claude.com/docs/en/best-practices) — the source behind several principles here, including the "what you can stop doing" table.
- [Changelog](https://code.claude.com/docs/en/changelog) · [What's new](https://code.claude.com/docs/en/whats-new) — start a [refresh pass](../shared/workflows/doc-freshness.md) here.
- [Glossary](https://code.claude.com/docs/en/glossary) · [The `.claude` directory](https://code.claude.com/docs/en/claude-directory).

## Out of scope

- **Enterprise and admin** — [IAM](https://code.claude.com/docs/en/iam), [team](https://code.claude.com/docs/en/team), [admin setup](https://code.claude.com/docs/en/admin-setup), [cloud providers](https://code.claude.com/docs/en/bedrock-vertex), [corporate proxy](https://code.claude.com/docs/en/corporate-proxy), [GitHub Enterprise](https://code.claude.com/docs/en/github-enterprise-server). Provider caveats get captured inline where they change a documented behaviour, never as standalone docs.
- **JetBrains** — VS Code only.
- **Agent SDK** — on `platform.claude.com`. Relevant only when building a custom agent on top of Claude Code.
