---
verified_against: 2.1.308
verified_on: 2026-10-27
sources:
  - https://code.claude.com/docs/en/memory
  - https://code.claude.com/docs/en/settings
  - https://code.claude.com/docs/en/permissions
---

# Setting up Claude Code from another project (agent-runnable)

> **You are the agent reading this.** The user is in another project and asked you to look at this playbook and set up the Claude Code basics they need. This doc is the literal recipe. Follow it in order. Stop and ask only at the decision points marked **Confirm with user**.

This doc is the agent-runnable counterpart to [`new-project.md`](new-project.md). If a *human* is reading this and they're in their own project, send them to [`new-project.md`](new-project.md) instead.

## Pre-flight: read the target project

Before writing anything, gather context:

1. **Read the target project's root files**: `README.md`, `CLAUDE.md` (if it exists), whichever build manifest is present, `.env.example`, top-level `docs/` if present.
2. **List `.claude/`** in the target if it exists: `ls -la .claude/ 2>/dev/null` — note any existing `settings.json`, `agents/`, `skills/`, `rules/`, `hooks/`.
3. **Take the real commands from the manifest.** Read the scripts block, plugin config, or task definitions — not the ecosystem's conventional commands. Projects override those constantly.

   You need: install, build, test, lint, run. Note which are genuinely absent rather than inventing them.
4. **Identify the secrets / sensitive paths** the project uses: read `.env.example`, `.gitignore`, and any `secrets/` or `credentials/` dirs.
5. **Identify any hard constraints** in the README or CLAUDE.md — version locks, a service that must be running, anything marked MUST or NEVER.

Write a brief one-paragraph summary of what you found before proceeding.

## Step 1: `CLAUDE.md` handling

**If `CLAUDE.md` already exists:**

- **Do not overwrite it.** It is the team's source of truth.
- Read it carefully. Note what's already covered.
- Identify any gaps relevant to common Claude Code patterns: build commands, test commands, lint commands, architecture pointers, hard "always/never" rules.
- If you find clear gaps, **propose additions** (don't write them yet) and **confirm with the user** before editing. Show the proposed additions in your response.

**If `CLAUDE.md` does NOT exist:**

- Generate a starter and write it to `./CLAUDE.md`.
- Use the template below. Adapt the placeholders to what you observed in the target project.
- Keep it under 200 lines. Aim for ~60-100.

### `CLAUDE.md` starter template

```markdown
# CLAUDE.md

[1-2 sentences: what this project is, from README + observed code.]

## Build, test, run

- Install deps: `[the command the manifest actually defines]`
- Run tests: `[detected command]`
- Lint: `[detected command if present]`
- Run locally: `[detected command if present]`

## Conventions

- [Language/runtime version, if locked]
- [Naming conventions observed in src/]
- [Error handling patterns observed]
- [Test layout — co-located vs separate dir]

## Architecture

- [Where the main entry point is]
- [Top-level module map: src/X handles A, src/Y handles B]
- [External integrations: APIs, databases, MCP servers, etc.]

## Hard constraints

- [Anything in README marked as MUST/NEVER]
- [Version locks with reasons]
- [Platform-specific concerns]

## Workflow

- [Branch strategy if observable from git log]
- [Commit message convention if observable]
- [PR/review expectations if mentioned]
```

Fill in only what you actually observed. **Don't invent conventions.** If you can't tell whether tests live in `tests/` or alongside source files, leave that line out rather than guessing.

After writing, summarize what's in the new `CLAUDE.md` so the user can correct anything you misread.

## Step 2: `.claude/settings.json` (committed)

Create `.claude/settings.json` with deny rules for secrets and high-blast-radius commands. **If the file already exists, read it first and propose additions rather than overwriting.**

### Universal denies (use for all project types)

```json
{
  "permissions": {
    "defaultMode": "default",
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(./credentials*)",
      "Bash(curl http*)",
      "Bash(wget *)",
      "Bash(git push * main)",
      "Bash(git push * master)",
      "Bash(git push --force *)",
      "Bash(git push -f *)"
    ],
    "ask": [
      "Bash(rm -rf *)"
    ]
  }
}
```

### Add what this project actually needs

The universal block covers secrets and destructive git. Beyond it, add rules only for what you observed:

- **A publish or deploy command** in the manifest (`npm publish`, `mvn deploy`, `cargo publish`, …) → add to `ask`.
- **An install command that isn't lockfile-aware** → add to `ask`, and put the lockfile-aware one in the allow list in Step 3.
- **Generated or vendored directories** the project treats as read-only → add to `deny`.

Don't add rules for commands the project doesn't use. A deny list full of irrelevant entries is noise that makes the real rules harder to review.

### Project-specific secrets

After applying the templates, **add any secrets paths you observed in `.env.example` or `.gitignore`**. Examples:
- If `.env.example` shows `STRIPE_KEY=`, the existing `Read(./.env*)` covers it.
- If you saw `credentials/aws.json` in the project, add `Read(./credentials/aws.json)`.
- If the project has a non-standard secrets dir like `keys/`, add `Read(./keys/**)`.

## Step 3: `.claude/settings.local.json` (gitignored — personal allows)

Create `.claude/settings.local.json` with the common safe-command allows for the project type. The file is gitignored automatically when Claude Code creates it.

### Universal personal allows

```json
{
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git add *)",
      "Bash(git commit *)",
      "Bash(git branch *)",
      "Bash(git checkout *)"
    ]
  }
}
```

### Add this project's build and test commands

Take them from the manifest you read during pre-flight, exactly as the project defines them:

```json
"Bash(<test command> *)",
"Bash(<build command> *)",
"Bash(<lint command> *)"
```

Allow the commands that are safe to run unattended and that you'll run often — test, build, lint, typecheck, and the lockfile-aware install. Leave anything that publishes, deploys, or mutates shared state out of the allow list.


## Step 4: What NOT to do

These are deliberate **don'ts** for an agent applying this playbook:

- **Don't overwrite existing `CLAUDE.md`.** It is team-shared. Propose additions and confirm.
- **Don't overwrite existing `.claude/settings.json`.** Merge in proposed denies, but read the existing file first and check for conflicts.
- **Don't set `permissions.defaultMode` to `auto` or `bypassPermissions` in `.claude/settings.json`.** It's ignored from project settings as a repo-spoof guard — and even if it weren't, those modes are personal-scope decisions.
- **Don't auto-create subagents** in `.claude/agents/`. Subagents are deferred until the rule-of-three friction shows up. See [`../principles/defer-until-friction.md`](../principles/defer-until-friction.md).
- **Don't auto-create skills** in `.claude/skills/`. Same reason.
- **Don't auto-create hooks** in `settings.json`. Same reason — and hooks have non-obvious failure modes you shouldn't introduce without the user's awareness.
- **Don't add MCP servers** unless the user explicitly asks. MCP servers consume context every session.
- **Don't add `.claude/settings.local.json` to `.gitignore` manually if Claude Code created the file** — it handles that automatically. Only add a gitignore line if you created the file yourself outside Claude Code.

## Step 5: Report back to the user

After you've applied the steps, write a summary in your response with this structure:

```
## Setup summary

**Stack detected:** [language/framework, and the manifest you read it from]

**Files written:**
- CLAUDE.md — [created from template / additions proposed (not written) / left untouched because already exists]
- .claude/settings.json — [created / merged with existing]
- .claude/settings.local.json — [created]

**What CLAUDE.md captures:**
[1-2 line summary]

**Deny rules applied:**
[Bullet list of the key categories: secrets, force-push, etc.]

**Allow rules applied (personal):**
[Bullet list of the key categories: git read, [project-type] build/test commands]

**Things I deferred (rule of three — see shared/principles/defer-until-friction.md):**
- Custom subagents
- Custom skills
- Hooks
- MCP servers

**Things to confirm:**
1. [Anything you guessed during CLAUDE.md generation]
2. [Any project-specific path you weren't sure about]
3. [Any constraint from the README you weren't sure how to enforce]

**Next steps the user should consider:**
- Set `~/.claude/settings.json` with `"permissions": { "defaultMode": "plan" }` if not already set (this is a per-machine personal preference; can't be set from project scope).
- Run `/status` to confirm the new settings loaded.
- Run `/permissions` to review and adjust the deny rules.
```

## When you're done

Stop here. **Do not** continue adding scaffolding speculatively. The user's project has the foundational Claude Code setup. Anything beyond this comes from observed friction in their actual sessions — covered by [`new-project.md`](new-project.md) → "Optional scaffolding" and [`../../claude/workflows/scaffolding-decision.md`](../../claude/workflows/scaffolding-decision.md).

## See also (for the agent's reference, if needed)

- [`new-project.md`](new-project.md) — the human-oriented version of this checklist with more detail on each step.
- [`new-machine.md`](new-machine.md) — once-per-machine setup; only relevant if the user is on a fresh machine.
- [`../principles/defer-until-friction.md`](../principles/defer-until-friction.md) — the rule-of-three rationale for everything you DIDN'T scaffold.
- [`../../claude/config/memory.md`](../../claude/config/memory.md) — full `CLAUDE.md` mechanics if needed for advanced cases.
- [`../../claude/config/permissions.md`](../../claude/config/permissions.md) — full permission rule syntax.
