# Setting up Claude Code from another project (agent-runnable)

> **You are the agent reading this.** The user is in another project and asked you to look at this playbook and set up the Claude Code basics they need. This doc is the literal recipe. Follow it in order. Stop and ask only at the decision points marked **Confirm with user**.

This doc is the agent-runnable counterpart to [`new-project.md`](new-project.md). If a *human* is reading this and they're in their own project, send them to [`new-project.md`](new-project.md) instead.

## Pre-flight: read the target project

Before writing anything, gather context:

1. **Read the target project's root files**: `README.md`, `CLAUDE.md` (if it exists), `pyproject.toml` / `package.json` / `go.mod` / `Cargo.toml` / etc., `.env.example`, top-level `docs/` if present.
2. **List `.claude/`** in the target if it exists: `ls -la .claude/ 2>/dev/null` — note any existing `settings.json`, `agents/`, `skills/`, `rules/`, `hooks/`.
3. **Detect project type** from files present:

| File present | Project type | Settings template to use |
| --- | --- | --- |
| `pyproject.toml`, `requirements.txt`, `setup.py` | **Python** | "Python template" below |
| `package.json` | **Node / JS / TS** | "Node template" below |
| `go.mod` | **Go** | "Go template" below |
| `Cargo.toml` | **Rust** | "Rust template" below |
| `Gemfile` | **Ruby** | "Generic template" + adapt |
| `pom.xml`, `build.gradle` | **Java / Kotlin** | "Generic template" + adapt |
| None of the above | **Generic** | "Generic template" below |

4. **Identify the secrets / sensitive paths** the project uses: read `.env.example`, `.gitignore`, and any `secrets/` or `credentials/` dirs.
5. **Identify any platform constraints** in the README or CLAUDE.md (Python version locks, Windows-only modules, etc.).

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

- Install deps: `[detected command — e.g., uv sync, npm install, go mod download]`
- Run tests: `[detected command]`
- Lint: `[detected command if present]`
- Run locally: `[detected command if present]`

## Conventions

- [Python version / Node version / etc. if locked]
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

### Project-type-specific additions

Add the matching block to the `deny` array. For multiple matches (e.g. Python + Node monorepo), add both.

**Python template additions:**
```
"Read(./*.pem)",
"Read(./venv/**)",
"Bash(pip install --upgrade pip)"
```
And to `ask`:
```
"Bash(pip install *)"
```
*(Reason: lockfile-aware install commands like `uv sync` should still pass through — add them to `.claude/settings.local.json` allow list. See Step 3.)*

**Node template additions:**
```
"Read(./.env.local)",
"Read(./.env.production)",
"Read(./.env.development.local)"
```
And to `ask`:
```
"Bash(npm publish *)",
"Bash(pnpm publish *)"
```

**Go template additions:**
```
"Read(./*.pem)"
```
And to `ask`:
```
"Bash(go install *)"
```

**Rust template additions:**
```
"Read(./target/.fingerprint/**)"
```
And to `ask`:
```
"Bash(cargo publish *)"
```

**Generic template:** the universal block alone is fine. Add project-specific paths the user identifies during pre-flight.

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

### Project-type-specific allows

Add the matching block to `allow`:

**Python:**
```
"Bash(uv *)",
"Bash(uv sync*)",
"Bash(uv run *)",
"Bash(pytest *)",
"Bash(python -m *)",
"Bash(ruff *)",
"Bash(mypy *)"
```

**Node:**
```
"Bash(npm run *)",
"Bash(npm test*)",
"Bash(npm install)",
"Bash(npm ci)",
"Bash(npx *)",
"Bash(node *)",
"Bash(pnpm run *)",
"Bash(pnpm install)"
```

**Go:**
```
"Bash(go test *)",
"Bash(go build *)",
"Bash(go run *)",
"Bash(go vet *)",
"Bash(go fmt *)",
"Bash(go mod *)"
```

**Rust:**
```
"Bash(cargo test *)",
"Bash(cargo build *)",
"Bash(cargo run *)",
"Bash(cargo check *)",
"Bash(cargo clippy *)",
"Bash(cargo fmt *)"
```

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

**Project type detected:** [Python / Node / etc.]

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
