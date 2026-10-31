# Enforcement vs. influence

Instructions in memory files and prompts are **influence** the model may follow. Permissions and hooks are **enforcement** the model cannot bypass. Pick the layer that matches the guarantee you need.

Anthropic frames the same split as **non-deterministic** (you write it, the model reads it, the model decides) versus **deterministic** (you configure it, the harness executes it every time). Most capabilities exist in both forms:

| | Non-deterministic — the model decides | Deterministic — the harness enforces |
| --- | --- | --- |
| What to do | Your prompt, memory files, skills the model picks up itself | Hooks; skills only you can start |
| What it can touch | The permission classifier | Deny rules, the sandbox, managed settings |
| When it's done | A request for evidence, a goal condition, a review | Tests, typecheck, a `Stop` hook, CI |
| How much it spends | Effort level, the model's own pace | Turn and budget caps, a spend limit |
| Who has the last word | The model's summary of what it did | Branch protection, the person who approves the merge |

**Default to the non-deterministic side.** Almost every guideline has exceptions, and a model with the actual code in front of it judges whether a guideline fits better than a rule you wrote in advance. Reach for enforcement when something must *always* or *never* happen regardless of that judgement.

Two signals you've picked the wrong side:

- You keep adding emphasis to an instruction and it still gets missed → it belongs in a hook.
- You have hooks enforcing things that are only preferences → delete them and let the model decide.

## Why it's true

Memory files (`CLAUDE.md`, `.claude/rules/`, custom system prompts) are sent to the model as context. The model reads them and tries to follow them, but there's no code-level check that the model actually does. If the model decides a rule doesn't apply or just forgets, the work proceeds.

Permissions are evaluated by Claude Code itself before tool calls execute, so a deny rule holds regardless of what the model thinks. Rules are evaluated deny → ask → allow, first match wins.

What a rule covers is narrower than it looks, though: a `Read(./.env)` deny stops the Read tool and a `cat .env`, but not a recursive `grep` that never names the file. Enforcement is only as good as the surface the rule reaches — which is why the sandbox sits below it on this ladder.

Hooks run as code in your shell. A `PreToolUse` hook exiting with code 2 blocks the tool call before permission rules are even evaluated. The hook can inspect the full tool input (the exact Bash command, the exact file path, the model the subagent is using) and make a decision your shell logic controls.

Sandboxing enforces at the OS level (Seatbelt on macOS, bubblewrap on Linux/WSL2). It applies to every subprocess the agent spawns, including arbitrary Python scripts or Node tools, not just the agent's direct tool calls.

## The decision pattern

| You want to say… | Use |
| --- | --- |
| "Please try to do X" | Memory (CLAUDE.md / rules) |
| "Always do X at moment M" | Hook (PostToolUse, etc.) |
| "Never do X" | Permission `deny` OR hook |
| "Allow X except in case Y" | Hook (permissions are first-match-wins, no allowlist exceptions to a deny) |
| "No process anywhere can touch X" | Sandbox (OS-level) |

## How to apply

- **Don't write "NEVER push to main" in CLAUDE.md and consider it done.** Add `Bash(git push * main)` (and `master`) to `permissions.deny`. Memory is influence; the model may convince itself an exception applies. The permission rule won't.
- **For "must run at moment X" automation** — auto-format after every edit, run lint before commit, log every database write — use a hook, not memory. Memory might get the agent to remember; a hook makes it deterministic.
- **For nuanced "allow X except Y"** rules (e.g. "any git command except force push to main"), use a `PreToolUse` hook that inspects the command. Permission rules are first-match-wins without specificity unwinding — a broad deny eats narrower allows. The hook can express the conditional logic permissions can't.
- **For "no subprocess can read this file"** (e.g. a Python script the agent runs shouldn't read credentials either), enable sandbox filesystem rules. Permission rules only cover the agent's direct file tools, not arbitrary subprocesses.

## Example

Goal: keep the agent away from `.env` files.

| Layer | What it looks like | What it actually prevents |
| --- | --- | --- |
| CLAUDE.md | "Never read .env files." | Hopeful. The agent will probably listen. May not in edge cases. |
| `permissions.deny: ["Read(./.env)"]` | Built-in deny rule | Agent's Read tool cannot read it, nor an `@file` mention or IDE-shared context. Covers Bash file commands that **name** the file (`cat`, `head`, `sed`, `tee`) and redirection targets. **Does not cover a command that reads without naming the file**, such as a recursive `grep`. |
| `PreToolUse` hook on Bash | Script blocks `cat .env`, `grep secret .env`, etc. | Bash invocations the permission rules miss (process wrappers, weird syntax). |
| `sandbox.filesystem.denyRead: [".env"]` | OS-level rule | A Python script the agent runs that calls `open('.env')` is blocked too. |

Each lower layer catches what the upper layer misses. For most projects, permission denies cover the practical case. For sensitive client work, layer the sandbox underneath.

## A note on tradeoffs

Enforcement layers are stricter but more expensive to maintain:

- Memory: cheap to write, cheap to change, no friction.
- Permission rules: cheap to write, fast to add, occasional false positives.
- Hooks: real work to write (script + edge cases) and maintain (output protocol, exit codes).
- Sandbox: minimal per-rule cost but you have to think about every subprocess that might run.

Don't reach for hooks when permissions would do. Don't reach for permissions when memory is enough for the situation. **Use the lightest enforcement layer that gives you the guarantee you need** — and reach for a stricter one when the lighter one demonstrably failed.

## See also

- [`claude/config/permissions.md`](../../claude/config/permissions.md) — Claude Code's permission system in detail.
- [`claude/tools/hooks.md`](../../claude/tools/hooks.md) — the full hooks reference.
- [`claude/config/memory.md`](../../claude/config/memory.md) — what memory is and isn't.
- [`durability-ladder.md`](durability-ladder.md) — the related axis of "how long does this rule need to last."
- [Deterministic and non-deterministic instructions](https://code.claude.com/docs/en/best-practices) — the source for the framing above.
