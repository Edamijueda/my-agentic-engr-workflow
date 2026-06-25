# Context is finite

Every token of context costs — time, money, and the agent's attention. What you load is what you spend.

## Why it's true

Modern coding agents have finite context windows (Claude Code: 200K by default, 1M on some models). That budget is shared across:

- The system prompt and tool definitions.
- Project context (memory files, path-scoped rules, auto memory).
- The conversation so far.
- Every file the agent reads.
- Skill bodies once invoked.
- Hook output that the hook explicitly forwards to the model.

When the window fills, the session compacts: history gets summarized, file contents are replaced with the summary, intermediate reasoning is dropped. The agent can still reference work but loses the exact code it read earlier.

Beyond the absolute cap, context dilutes attention. A 200-line memory file gets more reliable adherence than an 800-line one. Long files burn tokens AND reduce the chance the agent follows the rules.

## How to apply

- **Be specific in prompts.** "Fix the bug in `auth.ts` line 42" beats "fix the auth bug." Specificity stops the agent from reading three files when one would do. File reads dominate context usage during work.
- **Delegate research to a subagent.** A subagent reads files in its own context window. You get back a summary; the verbose reads stay isolated. For Claude Code: the built-in `Explore` subagent is read-only and cheap (Haiku model). See [`claude/tools/subagents.md`](../../claude/tools/subagents.md).
- **Keep memory files tight.** Aim for under 200 lines. Split long files into path-scoped rules so they only load when relevant (Claude Code: `.claude/rules/*.md` with `paths:` frontmatter). See [`claude/config/memory.md`](../../claude/config/memory.md).
- **Hide side-effect skills from auto-discovery.** Skills with `disable-model-invocation: true` don't appear in the model's skill listing — zero context cost until you `/invoke` them by name. Use for `/commit`, `/deploy`, etc.
- **Prefer "return a summary" over "show me everything."** When asking the agent to investigate, ask for what you actually need: "list the three files with the most TODOs" beats "show me all TODOs in the project."

## Example

You want to understand how authentication works in a new codebase.

| Approach | Token cost in main context |
| --- | --- |
| "Explain auth in this codebase" → agent reads `auth.ts`, `middleware.ts`, `session.ts`, `tokens.ts`, runs grep | ~6K tokens of file content + ~500 tokens of analysis |
| "Use the Explore subagent to research auth and summarize" | ~400 tokens (the summary) returns to your main context. The 6K of files stays in the subagent. |

Same answer, ~15× less main-context cost.

## See also

- [`claude/tools/context-window.md`](../../claude/tools/context-window.md) — what fills the budget and what survives `/compact`.
- [`claude/tools/prompt-caching.md`](../../claude/tools/prompt-caching.md) — when the cache wins and when it gets invalidated.
- [`durability-ladder.md`](durability-ladder.md) — where to put a rule so it survives the right boundaries.
