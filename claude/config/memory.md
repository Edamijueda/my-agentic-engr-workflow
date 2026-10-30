---
verified_against: 2.1.308
verified_on: 2026-10-27
sources:
  - https://code.claude.com/docs/en/memory
---

# Memory: CLAUDE.md and auto memory

How Claude carries knowledge across sessions. Distilled from `https://code.claude.com/docs/en/memory` (fetched via paste; verify against source).

## Mental model

Two systems, both loaded at session start, both treated as **context — not enforced configuration**:

| | CLAUDE.md | Auto memory |
| --- | --- | --- |
| Who writes it | You | Claude |
| What it contains | Instructions and rules | Learnings and patterns |
| Scope | Project / user / org | Per repository (shared across worktrees) |
| Loaded into | Every session, in full | Every session, first 200 lines or 25KB |
| Use for | Coding standards, workflows, project architecture | Build commands, debug insights, preferences Claude discovers |

If you need a hard "must / must not" guarantee, use a [PreToolUse hook](https://code.claude.com/docs/en/hooks-guide) or `permissions.deny` in settings instead. CLAUDE.md is influence, not enforcement.

## CLAUDE.md scope ladder

Four locations, loaded broadest → most specific. Later files appear *after* earlier ones in context (so more specific wins on conflicts).

| Scope | Location | Notes |
| --- | --- | --- |
| **Managed policy** | macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`<br>Linux/WSL: `/etc/claude-code/CLAUDE.md`<br>**Windows: `C:\Program Files\ClaudeCode\CLAUDE.md`** | Deployed by IT/DevOps. Cannot be excluded by `claudeMdExcludes`. |
| **User** | `~/.claude/CLAUDE.md` | Personal preferences across all projects. |
| **Project** | `./CLAUDE.md` or `./.claude/CLAUDE.md` | Team-shared via git. **This is what we have at the repo root.** |
| **Local** | `./CLAUDE.local.md` | Personal project-specific; `.gitignore` it. |

The walk is from filesystem root down to your CWD — so `foo/CLAUDE.md` loads before `foo/bar/CLAUDE.md`. Within a directory, `CLAUDE.local.md` is appended after `CLAUDE.md`.

Subdirectory `CLAUDE.md` files **under** the CWD are *not* loaded at launch — they load on demand when Claude reads files in those subdirs.

## Where each instruction belongs

Decision tree when you find yourself typing a rule:

1. **Must run at a specific moment** (e.g., before every commit) → [hook](https://code.claude.com/docs/en/hooks). Not memory.
2. **Must block a tool/path regardless of intent** → `permissions.deny` in settings. Not memory.
3. **Multi-step procedure or only relevant in one area** → [skill](https://code.claude.com/docs/en/skills) (loaded on demand) or a path-scoped rule.
4. **Always-on guidance for the whole project** → project `CLAUDE.md`.
5. **Always-on personal preference across all projects** → `~/.claude/CLAUDE.md`.
6. **Personal preference for one project, not for the team** → `./CLAUDE.local.md`.
7. **Org-wide policy** → managed policy `CLAUDE.md` + managed `settings.json`.

## `.claude/rules/` — modular and path-scoped

For projects too big for a single `CLAUDE.md`, drop topic files into `.claude/rules/` (loaded recursively). Without frontmatter, they load every session like `CLAUDE.md`. With `paths:` frontmatter, they only load when Claude touches matching files.

```markdown
---
paths:
  - "src/api/**/*.ts"
  - "tests/**/*.test.ts"
---

# API rules
- All endpoints must include input validation.
```

Symlinks are supported (point `.claude/rules/shared` at a shared dir for cross-project reuse). User-level rules live at `~/.claude/rules/` and apply to every project.

Pick rules over imports when you want path scoping; pick imports when you just want to split for organization (imported files still load in full at launch).

## Writing effective instructions

- **Size:** target **under 200 lines per file**. Longer files burn context *and* reduce adherence.
- **Structure:** markdown headers and bullets. Claude scans the same way humans do.
- **Specificity:** "Use 2-space indentation" beats "format code properly." "Run `npm test` before committing" beats "test your changes."
- **Consistency:** conflicting rules across files → Claude picks arbitrarily. Audit periodically.
- **HTML comments** (`<!-- … -->`) outside code blocks are **stripped before injection** — free space for human-only notes that don't burn tokens. Inside code blocks they're preserved.

## Imports

`@path/to/file` in any CLAUDE.md expands and loads that file at launch. Both relative and absolute paths work; relative resolves from the file containing the import. Imports can chain up to 4 hops deep. First time you launch a project with external imports, Claude shows an approval dialog.

To mention a path without importing, wrap it in backticks: `` `@README` `` is literal, `@README` imports.

Useful patterns:
- Cross-worktree personal instructions: `@~/.claude/my-project-instructions.md` (since `CLAUDE.local.md` only exists in the worktree where you created it).
- Pulling in existing docs: `See @README for overview and @package.json for npm commands.`

## AGENTS.md interop

Claude only reads `CLAUDE.md`. If a repo already uses `AGENTS.md` (or `.cursorrules`, `.devin/rules/`, `.windsurfrules`):

- Create `CLAUDE.md` that imports it: `@AGENTS.md` on its own line, then any Claude-specific additions.
- Or symlink: `ln -s AGENTS.md CLAUDE.md`. The `@AGENTS.md` import is the more portable choice.
- `/init` reads `AGENTS.md` and other tool configs when generating a starter `CLAUDE.md`.

## `/init` — generate a starter

Inside a session, `/init` analyzes the repo and writes a `CLAUDE.md`. If one already exists, it suggests improvements instead of overwriting.

Set `CLAUDE_CODE_NEW_INIT=1` to enable an interactive multi-phase flow that also sets up skills and hooks, explores via a subagent, and shows a reviewable proposal before writing.

## Auto memory

Auto memory is Claude writing notes for itself across sessions. On by default.

### Location

`~/.claude/projects/<project>/memory/` — `<project>` is derived from the git repo, so all worktrees share one dir. Outside a git repo, project root is used.

```text
~/.claude/projects/<project>/memory/
├── MEMORY.md          # Index. First 200 lines or 25KB loaded into every session.
├── debugging.md       # Topic files. Loaded on demand only.
└── ...
```

**Resolved** (checked on v2.1.308): the `<project>` segment is path-encoded from the **repo root**, not from your CWD — e.g. `-Users-you-developer-my-project`. So the docs' "derived from the git repository" holds for the case they mean: every worktree and subdirectory of one checkout shares a single memory dir. But two checkouts of the same repo at *different paths* get different dirs, and so does the same repo after you move it.

To pin the name, set `CLAUDE_CODE_PROJECT_DIR_NAME` alongside `CLAUDE_CONFIG_DIR` (v2.1.234+). Claude Code then uses that literal name under `<config dir>/projects/` whatever repo you launch in, so every project started with that config dir shares one memory dir.

Note what this does and doesn't buy you. Memory files are **not** synced across machines by Claude Code — that's true regardless. A pinned name only guarantees that if *you* sync the directory (or point `CLAUDE_CONFIG_DIR` at something already synced), both machines resolve to the same folder instead of silently diverging on two path-derived names.

### Controls

| Knob | Where | Effect |
| --- | --- | --- |
| Toggle on/off (per project) | `autoMemoryEnabled` in `settings.json` | `false` disables. |
| Toggle on/off (interactive) | `/memory` command, auto-memory toggle | Per session. |
| Disable everywhere | `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` env var | Global kill switch. |
| Custom location | `autoMemoryDirectory` in `settings.json` | Absolute path or `~/`-prefixed. Project-scoped values require workspace trust. |

### What loads when

- `MEMORY.md`: first 200 lines / 25KB at session start.
- Topic files: only when Claude opens them via Read.
- This 200-line cap is **specific to `MEMORY.md`**. CLAUDE.md has no hard limit, just adherence degradation past ~200 lines.

Auto memory files are plain markdown — edit or delete any time. Use `/memory` to browse and open.

When Claude writes a memory file that starts with YAML frontmatter, Claude Code stamps the write time into a `modified` field as an ISO 8601 timestamp — so both you and Claude can see how current a given fact is on read-back.

## `/memory` command

Inside a session, `/memory` shows:
- Every CLAUDE.md, CLAUDE.local.md, and rules file currently loaded.
- Auto memory toggle.
- Link to open the auto memory folder.

Selecting a file opens it in your editor. When you tell Claude "remember that X", it writes to auto memory by default. To put it in CLAUDE.md instead, say "add this to CLAUDE.md" or edit via `/memory`.

## `/compact` behavior

- Project-root `CLAUDE.md` is **re-read from disk and re-injected** after `/compact`. Always survives.
- Nested CLAUDE.md files in subdirs are not re-injected automatically — they reload when Claude next opens a file there.
- Conversation-only instructions are lost on compaction. If something matters across the session, put it in CLAUDE.md.

## Settings reference (memory-related)

| Key | Scope | Purpose |
| --- | --- | --- |
| `claudeMd` (string) | Managed / policy only | Embed managed-policy CLAUDE.md content directly in `managed-settings.json`. Loads before user and project CLAUDE.md. Ignored in user/project/local settings. |
| `claudeMdExcludes` (array of globs) | Any | Skip specific CLAUDE.md files by path/glob. Managed-policy files cannot be excluded. Useful in monorepos. |
| `autoMemoryEnabled` (bool) | Any | Toggle auto memory. |
| `autoMemoryDirectory` (string) | Any | Custom auto-memory location. Project-scoped values gated by workspace trust. |

Env vars:
- `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` — disable auto memory globally.
- `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` — also load `CLAUDE.md`/`.claude/CLAUDE.md`/`.claude/rules/`/`CLAUDE.local.md` from dirs added via `--add-dir`.
- `CLAUDE_CODE_NEW_INIT=1` — enable the interactive multi-phase `/init` flow.

CLI flag worth knowing: `--setting-sources` (excluding `local` skips `CLAUDE.local.md` from additional dirs).

## Debugging

- `/memory` first — if a file isn't listed, Claude can't see it.
- Use the `InstructionsLoaded` hook to log exactly which instruction files load, when, and why. Best tool for path-specific rule debugging.
- For instructions that must happen at a fixed moment, switch to a hook — CLAUDE.md is not enforcement.
- For system-prompt-level instructions, `--append-system-prompt` works but must be passed every invocation (scripts only, not interactive).

## Practical applications for this repo

- `CLAUDE.md` at the root already covers project context — keep it under 200 lines and review periodically.
- If a directory here grows large, split it into `.claude/rules/` files with `paths:` frontmatter so each loads only when Claude touches the files it describes.
- `user_profile.md` in `~/.claude/projects/.../memory/` is fine where it is — it's personal and won't sync to the public repo.
