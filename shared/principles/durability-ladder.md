# Durability ladder

Rules live at different durability layers. Pick the layer based on **how long the rule needs to persist** and **which boundaries it needs to survive**.

## The ladder

From least durable to most durable:

| Layer | Survives `/clear`? | Survives `/compact`? | Survives session restart? | Survives machine switch? |
| --- | --- | --- | --- | --- |
| Conversation message | No | Summarized | No | No |
| Auto memory (`MEMORY.md`) | Yes | Yes (re-injected) | Yes | No (machine-local) |
| Project `CLAUDE.md` | Yes | Yes (re-injected from disk) | Yes | Yes (if committed) |
| User `~/.claude/CLAUDE.md` | Yes | Yes | Yes | No (machine-local) |
| Project `.claude/settings.json` | Yes | Yes (hot-reloaded) | Yes | Yes (if committed) |
| Hook | n/a (runs as code) | n/a | Yes | Yes (if committed) |
| OS sandbox | n/a | n/a | Yes (per-session) | Configuration is per-machine |

Two axes to track: **time boundaries** (clear, compact, restart) and **physical boundaries** (machine, repo).

## How rules degrade across the ladder

- **Conversation messages** are the most ephemeral. `/clear` wipes them. `/compact` summarizes them — you keep the gist but lose the exact text.
- **Auto memory** (Claude's notes to itself) re-injects after `/compact` and reloads on session start. But it's stored under `~/.claude/projects/<project>/memory/` — **machine-local**. Switching to your Windows VM means starting auto memory from scratch.
- **`CLAUDE.md` at the project root** is re-read from disk and re-injected after `/compact`. So it survives compaction without you doing anything. Commit it to git and it follows the repo across machines.
- **Path-scoped rules** (`.claude/rules/*.md` with `paths:` frontmatter) and **nested `CLAUDE.md` files** are different — they load lazily when the agent touches matching files. They get **summarized away** by `/compact` and only reload the next time a matching file is read.
- **`settings.json`** (permissions, hooks, env, autoUpdatesChannel, etc.) is hot-reloaded mid-session. Edits apply immediately for permissions/hooks, but `model` and `outputStyle` need a restart.
- **Hooks** run as code, not context. They're not "in the conversation" — they fire on lifecycle events. Most durable runtime mechanism short of OS-level rules.
- **OS sandbox** rules are enforced by the kernel against every subprocess. Survives anything the agent does within the session.

## How to apply

- **"Just this task, then forget"** → say it in the prompt. Don't write it down.
- **"Remember this for the rest of this session"** → auto memory works (it survives `/compact`), or just keep saying it in chat.
- **"Always know this about this project, on any machine"** → project `CLAUDE.md`. Commit it.
- **"Always know this about my work, no matter the project"** → user `~/.claude/CLAUDE.md`. Personal scope, doesn't cross machines.
- **"Always enforce on this project"** → project `.claude/settings.json` (committed). Permissions, hooks, env vars.
- **"Always enforce on my machine, all projects"** → user `~/.claude/settings.json`.
- **"Survive even when an agent ignores its own rules"** → hooks (block at lifecycle) or sandbox (block at OS).

## The cross-machine boundary

Things that **automatically cross** from Mac to Windows VM (if you commit them):
- Project `CLAUDE.md`
- `.claude/settings.json`
- `.claude/rules/`
- `.claude/agents/`
- `.claude/skills/`
- `.claude/hooks/` (scripts) — but POSIX vs PowerShell shell semantics differ; see [`enforcement-vs-influence.md`](enforcement-vs-influence.md) and [`claude/tools/hooks.md`](../../claude/tools/hooks.md).

Things that **don't cross**:
- Auto memory (`~/.claude/projects/<project>/memory/`) — machine-local.
- Personal `~/.claude/CLAUDE.md` and `~/.claude/settings.json` — machine-local.
- Credentials (`~/.claude/.credentials.json` on Linux/Windows, Keychain on macOS) — re-auth on each machine.
- Prompt cache — per-machine + per-directory.

If you want your personal preferences to follow you across machines, manage them via a separate private dotfiles-style repo and `git pull` on each machine. See the README in this repo for the cross-OS scope statement.

## Example: a "use 2-space indentation" rule

| Decision | Implication |
| --- | --- |
| Type into chat every session | Hopeful, wastes session time. |
| Write in conversation, keep using session | Lost on `/clear` or restart. |
| Add to auto memory ("Claude, remember 2-space indent") | Survives until restart; machine-local. Loses on machine switch. |
| Add to project `CLAUDE.md` | Survives compaction. Follows repo across machines. **Right answer for project-shared rules.** |
| Add to `~/.claude/CLAUDE.md` | Personal default for all projects, but only on this machine. **Right answer for personal preferences.** |
| Add as `.editorconfig` + a hook that runs prettier | Hooks fire deterministically. Survives even if agent ignores memory. Overkill for a style rule unless your team enforces strictly. |

The right durability is the one that matches how long the rule actually needs to persist. Higher up the ladder costs more to set up. Don't put project-style rules in user-scope; don't put one-time clarifications in `CLAUDE.md`.

## See also

- [`claude/config/memory.md`](../../claude/config/memory.md) — scope ladder for `CLAUDE.md` and rules.
- [`claude/config/settings.md`](../../claude/config/settings.md) — scope ladder for settings.
- [`claude/tools/context-window.md`](../../claude/tools/context-window.md) — what survives `/compact` in detail.
- [`enforcement-vs-influence.md`](enforcement-vs-influence.md) — the related axis of "how strict do I need this rule to be."
