# Defer until friction

Don't pre-scaffold. Most projects need exactly three things on day one (memory file, deny rules, plan-mode default). Add the next layer only when concrete friction shows up.

## Why it's true

Every layer of customization has a cost beyond the time to write it:

- **Custom subagents** sit in the agent's skill/subagent listing every session, eating description tokens whether you use them or not.
- **Hooks** run on every matching tool call. A `PreToolUse` hook on `Bash` fires for every shell command, adding latency and risk of misfiring on cases you didn't anticipate.
- **Custom skills** with auto-discovery enabled enter the skill index at startup. Even ones you wrote "just in case" cost description tokens forever.
- **MCP servers** add tool definitions to the system prompt (or push tool-search overhead if deferred). Connection lifecycle has its own failure modes.
- **Plugins** bundle all of the above. Each one is a maintenance commitment.

Pre-scaffolding is also pre-deciding what your workflow looks like. The shape of friction you anticipate rarely matches the shape of friction you actually hit. Building infrastructure for problems you don't have yet means you've shaped the project around the wrong constraints.

## The rule of three

The shorthand for friction-driven scaffolding:

| Times you've hit it | What to do |
| --- | --- |
| 1st time | Just type the correction or do the work. |
| 2nd time | Notice. Think briefly about where this belongs (memory? skill? hook?). Don't act yet. |
| 3rd time | Now scaffold. You know the shape because you have three data points. |

If you've never hit a particular friction, you don't know its real shape. Three hits is the cheapest reliable signal.

## What to defer on day one

The temptation list. Resist all of these for a brand-new project:

- **Custom subagents.** Built-in `Explore` is read-only, Haiku-fast, and handles 90% of research. Wait until you've delegated the same kind of task three times manually.
- **Hooks.** Add when memory is being ignored on a rule that actually matters, not preemptively. A hook for "every git push" is reasonable only if you've watched the agent push to the wrong branch.
- **Custom skills.** Built-in skills (`/code-review`, `/debug`, `/run`, `/verify`) cover common workflows. Wait until you've typed the same multi-step procedure into chat 3+ times.
- **MCP servers.** Add when the project genuinely needs Jira/Drive/Slack access. Don't add them for "completeness."
- **Plugins beyond `skill-creator`.** Most plugins are useful in specific contexts. Don't install marketplace recommendations preemptively.
- **`auto` permission mode.** Powerful but a research preview. Stay on `plan` or `acceptEdits` until you have a concrete reason to want auto's reduced prompts.
- **Detailed `.claude/rules/` hierarchy.** Start with one `CLAUDE.md`. Split into path-scoped rules only when the file genuinely exceeds 200 lines.

## What's not deferrable

A few things ARE worth setting up day one, because the friction they prevent is the kind that bites silently:

- **Memory file** with build/test commands and conventions — without it, the agent re-asks every session.
- **Deny rules** for secrets, `.env*`, and high-blast-radius commands — without these you're betting on agent judgment for irreversible operations.
- **Plan-mode-first personal default** — without this you're approving edits one at a time when you haven't seen the whole plan.

These three pay for themselves immediately on every session. Everything else is speculative.

## When to escalate

You're ready to scaffold a layer when you can answer:

1. **What concrete friction does this address?** ("I've typed `cat .env | grep` and gotten denied three times" — concrete. "I might need to read env files" — not concrete.)
2. **What's the smallest scaffold that addresses it?** A skill is cheaper than a subagent. A permission allow rule is cheaper than a hook. Don't reach for the heaviest tool first.
3. **What's the cost if I'm wrong about the shape?** A skill is easy to delete. A hook that fires on every Bash call costs you latency until you remove it. A subagent description sits in context forever.

If you can't answer (1) with a concrete past example — defer.

## Example

A new TypeScript project. The temptation list:

- Write a `code-reviewer` subagent.
- Write a `commit-and-push` skill.
- Add a `PostToolUse` hook that runs prettier after edits.
- Add a `PreToolUse` hook that blocks dangerous git commands.
- Install the `skill-creator` plugin so we can write skills more easily.

The friction-driven version: do **none** of these on day one.

Two weeks in, here's what actually happened:

- You delegated review tasks to `Explore` and `general-purpose` subagents. Worked fine. **No custom code-reviewer needed.**
- You typed `commit and push my changes` three times. **Time to write the skill** (`disable-model-invocation: true`).
- Prettier never came up — the project uses `dprint` and the editor handles it. **Hook would have been wrong.**
- The agent tried to push to `main` once and got blocked by your deny rule. **No hook needed.**
- You never wrote a new skill. **`skill-creator` install would have been wasted setup.**

The friction-driven path produced one scaffolding decision (the commit-push skill) instead of five. Less maintenance, less context cost, and the one scaffold you added is one you'll actually use.

## See also

- [`shared/project-setup/new-project.md`](../project-setup/new-project.md) — the day-one checklist (three things only).
- [`enforcement-vs-influence.md`](enforcement-vs-influence.md) — pick the lightest enforcement layer that gives the guarantee.
- [`context-is-finite.md`](context-is-finite.md) — the cost side of premature scaffolding.
