---
verified_against: 2.1.308
verified_on: 2026-10-27
sources:
  - https://code.claude.com/docs/en/overview
---

# Doc coverage: what's distilled, what's left

Replaces the original tier ranking from 2026-06-22. Every issue that document suggested has shipped, so ranking what to read first is no longer the useful question. What matters now is what's covered, what's deliberately not, and what the next gap is.

For the mechanics of checking whether the distilled docs still hold, see [`../../../shared/workflows/doc-freshness.md`](../../../shared/workflows/doc-freshness.md).

## Covered

Each of these has a distilled doc carrying a freshness stamp. The stamp, not this table, is the authority on how current it is.

| Source page | Lives in |
| --- | --- |
| Memory | [`claude/config/memory.md`](../../../claude/config/memory.md) |
| Settings, Settings reference | [`claude/config/settings.md`](../../../claude/config/settings.md) |
| Permissions, Permission modes | [`claude/config/permissions.md`](../../../claude/config/permissions.md) |
| Context window | [`claude/tools/context-window.md`](../../../claude/tools/context-window.md) |
| Prompt caching, Model config | [`claude/tools/prompt-caching.md`](../../../claude/tools/prompt-caching.md) |
| Hooks, Hooks guide | [`claude/tools/hooks.md`](../../../claude/tools/hooks.md) |
| Tools reference | [`claude/tools/tools-reference.md`](../../../claude/tools/tools-reference.md) |
| Sub-agents | [`claude/tools/subagents.md`](../../../claude/tools/subagents.md) |
| Skills | [`claude/tools/skills.md`](../../../claude/tools/skills.md) |
| Slash commands, Commands | [`claude/slash-commands/built-in.md`](../../../claude/slash-commands/built-in.md) |
| Advanced setup, Authentication, Network config | [`claude/setup/`](../../../claude/setup/) |
| VS Code | [`claude/setup/vs-code.md`](../../../claude/setup/vs-code.md) |

## Next gaps, in order

### 1. Artifacts

The largest single hole. `Artifact` appears in the tools reference as a tool and nowhere else, but the surface around it is substantial: published pages, Artifact types, design systems, runtime capabilities, the shared artifact database, and comment threads. Nothing in the playbook says when publishing a page beats writing a file, which is the decision that actually comes up.

Needs its own issue, and probably its own doc rather than a section bolted onto the tools reference.

### 2. MCP

[MCP](https://code.claude.com/docs/en/mcp) is referenced from several docs but never distilled. Deferred originally because there was no client project needing it; that reasoning expires the moment one does. The context cost of a connected server is the part worth capturing — it's a per-session tax, and the tool-search behaviour that offsets it is already covered under prompt caching.

### 3. Agent teams and workflows

[Agent teams](https://code.claude.com/docs/en/agent-teams) and [workflows](https://code.claude.com/docs/en/workflows) didn't exist when the original sweep ran. Both bear on parallel work, which [`claude/workflows/parallel-with-forks.md`](../../../claude/workflows/parallel-with-forks.md) covers only via forks and subagents. Workflow fan-out has its own prompt-cache behaviour, already noted in the caching doc.

### 4. Best practices

[Best practices](https://code.claude.com/docs/en/best-practices) is Anthropic's own framing. Worth a read specifically to check it against [`shared/principles/`](../../../shared/principles/) — the value is in the disagreements, not the overlaps.

## Deliberately out of scope

Revisit only if the triggering condition becomes true.

| Area | Condition to revisit |
| --- | --- |
| Routines, scheduled tasks, GitHub Actions, GitLab CI | A project that benefits from scheduled or PR-triggered runs. `/schedule` and `/loop` are already in the command reference. |
| Desktop, web, Chrome, Slack, computer use | Adopting that surface. Partly covered already — mobility commands are in the command reference. |
| Devcontainer | A client project that ships one. |
| Agent SDK | Building a custom agent on Claude Code. |
| Enterprise (IAM, Team, admin, Bedrock, Google Cloud, Foundry, proxies, GHES) | Out of scope for solo work. Provider-specific caveats get captured inline where they affect a documented behaviour, not as standalone docs. |
| JetBrains | Out of scope — VS Code only. |

## Cross-OS debt

The original sweep flagged Windows specifics to capture during deep dives. Status:

- **Done:** install paths (PowerShell/CMD), Bash tool's dependency on Git for Windows, managed-settings paths, sandbox differences (Seatbelt vs. bubblewrap vs. none on native Windows), symlink elevation for `AGENTS.md`.
- **Open:** nothing known. The Windows VM has not been exercised against the current playbook since the refresh — treat unflagged Windows claims as verified by reading, not by running.
