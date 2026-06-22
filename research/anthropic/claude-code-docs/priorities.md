# Deep-dive priorities

Stated goals from `CLAUDE.md`: prepare a machine and project quickly, use tokens wisely, stay in the driver's seat. Ranking below reflects that — highest payoff first.

## Tier 1 — read first, distill into `claude/` immediately

1. **[Store instructions and memories](https://code.claude.com/docs/en/memory)** — `CLAUDE.md` is the cheapest, most portable lever. Already adopted at the repo root; need to capture scoping rules (project/user/org), import patterns, and best practices.
2. **[Settings](https://code.claude.com/docs/en/settings)** — `settings.json` is the central config (permissions, env, hot-reload). Foundational for hooks and permissions.
3. **[Permissions](https://code.claude.com/docs/en/permissions)** + **[Permission modes](https://code.claude.com/docs/en/permission-modes)** — direct token + safety control. Modes (default/plan/accept-edits/bypass) shape *how* sessions run; allow/deny rules shape *what* runs.
4. **[Explore the context window](https://code.claude.com/docs/en/context)** — direct token-cost lever. Understanding what eats context informs every other choice (subagents, plan mode, file reads).
5. **[Prompt caching](https://code.claude.com/docs/en/prompt-caching)** — biggest cost reduction per change in habit. Goes hand-in-hand with the context page.
6. **[Slash commands](https://code.claude.com/docs/en/slash-commands)** — in-session efficiency multiplier; custom commands compound over time.
7. **[Tools reference](https://code.claude.com/docs/en/tools-reference)** — knowing what's built in (and the output/size limits) directly affects token use.
8. **[Hooks](https://code.claude.com/docs/en/hooks)** — encodes "from now on, when X happens, do Y" in a way memory/preferences can't. High leverage once internalized.

## Tier 2 — read after Tier 1; builds on the above

9. **[Sub-agents](https://code.claude.com/docs/en/sub-agents)** — biggest single tool for protecting context window on long tasks.
10. **[Skills](https://code.claude.com/docs/en/skills)** — auto-invoked capabilities; pair well with hooks.
11. **[Best practices](https://code.claude.com/docs/en/best-practices)** — Anthropic's own framing; likely to validate or reshape Tier 1 takeaways.
12. **[Manage sessions](https://code.claude.com/docs/en/manage-sessions)** — resume/fork/share patterns matter once you're running parallel work or switching devices (Mac ↔ Windows VM).
13. **[Advanced setup](https://code.claude.com/docs/en/setup)** + **[Authentication](https://code.claude.com/docs/en/authentication)** — feeds the cross-OS install playbook in `claude/setup/`. Windows specifics already noted: PowerShell/CMD installer, Git for Windows required for Bash tool.
14. **[Visual Studio Code](https://code.claude.com/docs/en/vs-code)** + **[JetBrains IDEs](https://code.claude.com/docs/en/jetbrains)** — IDE integration details.
15. **[Explore the .claude directory](https://code.claude.com/docs/en/claude-directory)** — completes the mental model of project vs. global config.
16. **[Prompt library](https://code.claude.com/docs/en/prompt-library)** — reusable Anthropic-curated prompts; some likely belong in `claude/slash-commands/` or `shared/workflows/`.

## Tier 3 — useful, but not blocking

17. **[Common workflows](https://code.claude.com/docs/en/common-workflows)** — concrete recipes; some may slot into `shared/workflows/`.
18. **[Code Review](https://code.claude.com/docs/en/code-review)** — `/code-review` is already in the skill list; doc covers the deeper surface.
19. **[Sandboxing](https://code.claude.com/docs/en/sandboxing)** + **[Sandbox environments](https://code.claude.com/docs/en/sandbox-environments)** — relevant when running risky commands; **OS-specific** (Seatbelt on macOS, bubblewrap on Linux/WSL2 — note: native Windows has neither).
20. **[CLI reference](https://code.claude.com/docs/en/cli-reference)** + **[Interactive mode](https://code.claude.com/docs/en/interactive-mode)** — reference material; skim, don't deep-read.

## Tier 4 — defer / out of scope for now

- **[Routines](https://code.claude.com/docs/en/routines)**, **[GitHub Actions](https://code.claude.com/docs/en/github-actions)** — read once we have a project that benefits from scheduled or PR-triggered Claude runs.
- **[Desktop](https://code.claude.com/docs/en/desktop)**, **[Web](https://code.claude.com/docs/en/claude-code-on-the-web)**, Chrome extension, Computer use, Slack — different surfaces; revisit if/when adopted.
- **[Devcontainer](https://code.claude.com/docs/en/devcontainer)** — relevant if/when a client project ships with one.
- **Agent SDK** (on `platform.claude.com`) — only relevant when building a custom agent on top of Claude Code. Defer until that's an actual goal.
- **All enterprise pages** (IAM, Team, Admin setup, Bedrock/Vertex, Corporate proxy, GHES) — out of scope for solo/personal use.
- **[Discover plugins](https://code.claude.com/docs/en/discover-plugins)** — quick skim later to see if any community plugins are worth adopting.

## Cross-OS reminder

When we get to Tier 1 + Tier 2 deep dives, capture Windows-specific notes for: install (PowerShell/CMD), Bash tool dependency on Git for Windows, sandboxing differences (Seatbelt vs. bubblewrap vs. none on native Windows), and path conventions in `settings.json` / `CLAUDE.md`.

## Suggested next issues

- Tier 1 deep dive → `claude/config/` (memory, settings, permissions, permission modes), `claude/tools/` (tools, hooks, context window, prompt caching), `claude/slash-commands/` (slash commands).
- Tier 2 deep dive → `claude/setup/` (cross-OS install + auth + IDE), `claude/tools/` (subagents, skills), and `shared/workflows/` (manage sessions, prompt library extractions).
- Tier 3 → mostly contributes to `claude/workflows/` and `shared/workflows/`.
- Sister site → breadth pass on `platform.claude.com` (Agent SDK + API + prompt engineering) → feeds `shared/principles/`.
