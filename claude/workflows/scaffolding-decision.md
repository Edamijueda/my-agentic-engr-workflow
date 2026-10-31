# Scaffolding decision

You spotted friction worth scaffolding (rule of three from [`defer-until-friction`](../../shared/principles/defer-until-friction.md)). Now: **which mechanism do you reach for?**

## Decision tree

```
What's the shape of the friction?

├── "I keep typing the same context or correction into chat"
│   → CLAUDE.md (project root) or ~/.claude/CLAUDE.md (personal)
│
├── "Same context but only relevant when I'm in part X of the codebase"
│   → .claude/rules/<topic>.md with `paths:` frontmatter
│
├── "I keep invoking the same multi-step procedure"
│   → Skill (.claude/skills/<name>/SKILL.md)
│     With side-effects → also `disable-model-invocation: true`
│
├── "The agent keeps doing X when it shouldn't"
│   → permissions.deny: ["X(...)"] in .claude/settings.json
│
├── "Memory says don't X, but the agent still does it"
│   → PreToolUse hook with exit 2 (deterministic enforcement)
│
├── "I need a script to run at a specific lifecycle moment"
│   → Hook on PreToolUse / PostToolUse / SessionStart / etc.
│
├── "I keep delegating the same research-shaped task"
│   → Custom subagent (.claude/agents/<name>.md) with `memory: project`
│
├── "I want the agent to use an external service (Jira/Drive/Slack)"
│   → MCP server (.mcp.json or `mcpServers:` in subagent frontmatter)
│
├── "Multiple of the above bundled for a team"
│   → Plugin (deferred doc)
│
└── "I want this to never happen even if everything else fails"
    → Sandbox rule (sandbox.* in settings)
```

## Cost comparison

Sorted from cheapest to most expensive (per-session ongoing cost):

| Mechanism | Per-session cost | One-time write cost |
| --- | --- | --- |
| `permissions.deny` rule | ~0 tokens | 1 line of JSON |
| Skill with `disable-model-invocation: true` | 0 (until invoked) | The SKILL.md file |
| Hook | 0 context tokens (runs as code) | Script + JSON config + edge cases to handle |
| `.claude/rules/` with `paths:` | 0 (until matching files touched) | The rules file |
| CLAUDE.md content | ~50 tokens per line, every session | Just write it |
| Skill (default discoverable) | ~50-100 tokens (description) every session | The SKILL.md file |
| Custom subagent | Description tokens every session | Frontmatter + body |
| MCP server (deferred via tool search) | ~50-100 tokens (name) every session | Server setup |
| MCP server (eager) | Full tool definitions every session | Server setup |

> **The "free" tier**: deny rules, hidden skills, hooks. These have **zero ongoing context cost**. Reach for them first if they fit.

## Pair with the principles

Three principles back this decision:

### [`enforcement-vs-influence.md`](../../shared/principles/enforcement-vs-influence.md)
- Memory = influence (model may ignore).
- Permission rule = enforcement.
- Hook = enforcement that wins over permission rules.
- Sandbox = OS-level guarantee.

**Pick the lightest enforcement that gives the guarantee you need.**

### [`durability-ladder.md`](../../shared/principles/durability-ladder.md)
- Conversation < auto memory < CLAUDE.md < settings < hooks < OS sandbox.

**Pick the right layer for how long the rule needs to last and what boundaries it must cross.**

### [`defer-until-friction.md`](../../shared/principles/defer-until-friction.md)
- Rule of three before scaffolding.

**Most of the time, the right answer is "don't scaffold yet."**

## Worked examples

### Friction: "Agent doesn't know we use `bun test`, not `npm test`"

- Shape: agent is missing a fact.
- Cost: low — single fact, used always.
- Answer: **CLAUDE.md** (project root). One line: "Run tests with `bun test`."

### Friction: "Every PR I have to remind the agent about our error format"

- Shape: agent forgets context-specific convention.
- Specificity: only when editing `src/api/*`.
- Answer: **`.claude/rules/api-errors.md`** with `paths: ["src/api/**"]`.
- Why not CLAUDE.md? Saves context every session you're NOT working in `src/api/`.

### Friction: "Agent keeps trying to push to `main` directly"

- Shape: behavior to block.
- Answer: **`permissions.deny: ["Bash(git push * main)"]`** in `.claude/settings.json`.
- Why not memory? Memory is influence; the agent might decide "this case is different."
- Why not hook? Permission rule is simpler and sufficient. Hook would be overkill.

### Friction: "Agent runs prettier inconsistently after edits"

- Shape: needs to run at a specific moment (after edits).
- Answer: **`PostToolUse` hook** matching `Edit|Write`, runs prettier.
- Why not memory? Memory says "remember to run prettier" — agent may forget. Hook is deterministic.

### Friction: "Every two days I ask the agent to commit + push my changes"

- Shape: same multi-step procedure, side effects, you want to invoke explicitly.
- Answer: **Skill** at `.claude/skills/commit-push/SKILL.md` with `disable-model-invocation: true` and `allowed-tools: Bash(git add *) Bash(git commit *) Bash(git push *)`.
- Why hidden? Don't want Claude deciding to commit because your code "looks ready."

### Friction: "I keep asking the agent to audit our React component conventions"

- Shape: recurring research task with the same framing.
- Answer: **Custom subagent** at `.claude/agents/react-component-reviewer.md` with `memory: project`.
- Why subagent? The conventions evolve; memory captures what you've taught the agent over time. See [`custom-subagent-creation.md`](custom-subagent-creation.md).

### Friction: "Want the agent to read tickets from Jira"

- Shape: external service access.
- Answer: **MCP server** for Jira.
- Cost note: MCP tool definitions sit in the system prompt every session (unless deferred via tool search). For a single Jira integration that's worth it; weigh before adding many MCP servers.

### Friction: "Want the team to have all of the above"

- Shape: bundled customization for distribution.
- Answer: **Plugin** with skills/agents/hooks/MCP servers bundled.
- Deferred doc — see Anthropic's plugin docs for now.

## Anti-patterns

- **Writing a hook when memory would do.** Hooks cost script-maintenance time. Use them when memory has demonstrably failed (the friction example above where memory says "don't push to main" and the agent still does).
- **Writing a subagent for a one-off research task.** Just use built-in `Explore`. Defer.
- **Adding an MCP server "because we might need Jira sometime."** MCP servers sit in context. Wait until you actually need them.
- **Putting personal preferences in `.claude/settings.json`.** That's committed. Put them in `.claude/settings.local.json` (gitignored) or `~/.claude/settings.json` (personal).
- **Reaching for the strictest mechanism first.** Try the lightest one. Escalate when it fails.

## See also

- [`../../shared/principles/defer-until-friction.md`](../../shared/principles/defer-until-friction.md) — rule of three.
- [`../../shared/principles/enforcement-vs-influence.md`](../../shared/principles/enforcement-vs-influence.md) — pick the lightest enforcement.
- [`../../shared/principles/durability-ladder.md`](../../shared/principles/durability-ladder.md) — pick the right durability.
- [`custom-subagent-creation.md`](custom-subagent-creation.md) — once you've decided on a subagent.
- [`memory-curation.md`](memory-curation.md) — once you've decided on memory.
