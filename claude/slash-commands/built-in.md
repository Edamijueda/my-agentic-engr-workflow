# Built-in commands + bundled skills

Reference for the commands typed with `/` at the start of a message. Distilled from `https://code.claude.com/docs/en/commands`.

## Mental model

Three sources of `/` commands:

1. **Built-in commands** — coded into the CLI. Most things like `/clear`, `/model`, `/permissions`, `/help`.
2. **Bundled skills** — ship with Claude Code, work like skills you'd write yourself (a prompt handed to Claude). Marked **Skill** in the reference below. Disable globally with `disableBundledSkills: true`.
3. **Custom skills** — anything in `~/.claude/skills/`, `.claude/skills/`, or `.claude/commands/`. See `../tools/skills.md`.

> Type `/` to see what's available in your session. A command is **only recognized at the start** of your message. Text after the command name is passed as arguments.

> Not every command appears for every user. Availability depends on platform, plan, and environment (`/desktop` only on macOS/Windows with a Claude subscription; `/upgrade` only on Pro/Max).

> MCP servers can expose prompts as commands — `/mcp__<server>__<prompt>`. Dynamically discovered from connected servers.

## Commands across a typical workflow

The official doc groups commands by where they fit in a session. This is the most useful framing — the full alphabetical reference at the bottom is for lookups.

### First session in a repo

- `/init` — generate starter `CLAUDE.md`. `CLAUDE_CODE_NEW_INIT=1` for the interactive flow that also walks through skills, hooks, and personal memory.
- `/memory` — refine `CLAUDE.md`, toggle auto-memory, browse auto-memory entries.
- `/mcp` — connect MCP servers.
- `/agents` — set up subagents.
- `/permissions` — set allow/ask/deny rules. Alias `/allowed-tools`.

### During a task

- `/plan [description]` — switch to plan mode, optionally with the task already in hand.
- `/model [model]` — switch model. `s` in the picker = session-only switch. Confirmation dialog when conversation has prior output (next response re-reads full history uncached — see `../tools/prompt-caching.md`).
- `/effort [low/medium/high/xhigh/max/ultracode|auto]` — adjust reasoning. `auto` resets to model default. Immediate effect (doesn't wait for current response).
- `/context [all]` — visualize what's in context as a colored grid. Optimization suggestions for context-heavy tools, memory bloat.
- `/compact [instructions]` — free context by summarizing. Pass focus instructions to keep what matters. See `../tools/context-window.md` for what survives.
- `/btw <question>` — quick side question that doesn't add to conversation history.

### Parallel work

- `/agents` — open the subagent manager.
- `/tasks` (alias `/bashes`) — view background processes.
- `/background [prompt]` (alias `/bg`) — detach session to run as a background agent, freeing your terminal. `claude agents` to monitor.
- `/batch <instruction>` **(Skill)** — decompose change into 5-30 independent units, one background subagent per unit in an isolated worktree.
- `/fork <directive>` (v2.1.161+) — spawn a forked subagent that inherits the full conversation. Result returns when it finishes. (Before v2.1.161, `/fork` aliased `/branch`.)
- `/branch [name]` — branch the conversation at this point. Switches you into the branch; original recoverable via `/resume`.

### Before you ship

- `/diff` — interactive diff viewer. Left/right between current git diff and individual Claude turns. Up/down to browse files.
- `/code-review [low/medium/high/xhigh/max/ultra] [--fix] [--comment] [target]` **(Skill)** — review current diff for bugs + cleanups. `--fix` applies; `--comment` posts as inline GitHub PR comments; `ultra` runs deep cloud review.
- `/simplify [target]` **(Skill, v2.1.154+)** — cleanup-only review (no bug hunting). 4 parallel agents.
- `/review [PR]` — same engine as `/code-review`, applied to a GitHub PR. No args = list open PRs.
- `/security-review` — analyze pending changes for vulnerabilities (injection, auth, data exposure).
- `/verify` **(Skill, v2.1.145+)** — build + run + observe your app to confirm a change works. Not just tests.

### Between sessions

- `/clear [name]` (aliases `/reset`, `/new`) — start a new conversation with empty context. Previous conversation stays available via `/resume`. Pass a name to label the previous in the picker.
- `/resume [session]` (alias `/continue`) — resume by ID/name, or open picker. Background sessions appear marked `bg` (v2.1.144+).
- `/rename [name]` — rename current session. No name = auto-generate from history.
- `/teleport` (alias `/tp`) — pull a Claude Code on the web session into this terminal. Needs claude.ai subscription.
- `/remote-control` (alias `/rc`) — make this session available for remote control from claude.ai.

### When something's wrong

- `/rewind` (aliases `/checkpoint`, `/undo`) — roll code and conversation back to a checkpoint, or summarize from a selected message.
- `/doctor` — diagnose install/settings. Press `f` to have Claude fix reported issues.
- `/debug [description]` **(Skill)** — enable debug logging mid-session and analyze.
- `/feedback [report]` (aliases `/bug`, `/share`) — report a bug with session context attached.

## Other useful commands

### Workspace

- `/add-dir <path>` — extend file access to another dir for this session. **Loads skills** from that dir's `.claude/skills/` (covered in `../tools/skills.md`).
- `/cd <path>` (v2.1.169+) — move the session to a new working directory. **Preserves prompt cache** (appends new CLAUDE.md as a message instead of rebuilding system prompt). Cd permission rules apply (see `../config/permissions.md`).

### Skill management

- `/skills` — list available skills. Sort by token count (`t`). Hide individual skills with `Space` to cycle states, `Enter` to save (writes `skillOverrides` to `.claude/settings.local.json`).
- `/reload-skills` (v2.1.152+) — re-scan skill/command dirs without restart.

### Plugin management

- `/plugin [subcommand]` — open menu, or pass `list` / `install` / `enable` / `disable`.
- `/reload-plugins [--force]` — apply pending plugin changes without restart. Warns + skips if reload would invalidate prompt cache (changing MCP tool definitions). `--force` to apply anyway.

### Session info

- `/status` — Settings interface, Status tab. Version, model, account, connectivity. Works while Claude is responding.
- `/config [key=value ...]` (alias `/settings`) — open Settings interface, or set settings directly. From v2.1.181: `key=value` works. From v2.1.182: named shorthands like `theme=dark`, `model=sonnet`. Also works in `-p` and via Remote Control. `/config --help` lists settable keys.
- `/usage` (aliases `/cost`, `/stats`) — session cost, plan limits, activity. On Pro/Max/Team/Enterprise: breakdown by skill, subagent, plugin, MCP server.
- `/hooks` — view configured hooks (read-only browser — see `../tools/hooks.md`).
- `/recap` — one-line session summary.
- `/insights` — analyzes your sessions for project areas, interaction patterns, friction points.

### Interactive features

- `/goal [condition|clear]` — Claude keeps working across turns until condition is met. `clear`/`stop`/`off` to remove.
- `/loop [interval] [prompt]` **(Skill)** (alias `/proactive`) — repeat a prompt while session stays open. No interval = Claude self-paces. No prompt = autonomous maintenance from `.claude/loop.md` if present.
- `/fast [on|off]` — toggle fast mode (Opus-only typically). See `../tools/prompt-caching.md` for cache implications.
- `/sandbox` — toggle sandbox mode (supported platforms only).

### UI

- `/theme` — change color theme. `auto` matches terminal background. Custom themes from `~/.claude/themes/` or plugins.
- `/tui [default|fullscreen]` — set renderer; relaunch with conversation intact.
- `/focus` — show only your last prompt, one-line tool-call summary, and final response. Persists. Fullscreen mode only.
- `/color [color|default]` — prompt bar color for this session. Syncs to claude.ai/code when Remote Control connected.
- `/statusline` — configure custom status line. Describe what you want, or run without args to auto-configure from your shell prompt.
- `/scroll-speed` — adjust mouse wheel scroll speed. Fullscreen only.

### Output

- `/copy [N]` — copy last assistant response. `N` for Nth-latest. Code-block picker on multi-block responses. Press `w` in picker to write to file (useful over SSH).
- `/export [filename]` — export conversation as plain text.

### Voice / mobile

- `/voice [hold|tap|off]` — toggle voice dictation. Requires claude.ai account.
- `/desktop` (alias `/app`) — continue session in Claude Code Desktop app. macOS/Windows with subscription.
- `/mobile` (aliases `/ios`, `/android`) — QR code to download mobile app.

### Admin / extras

- `/login`, `/logout` — Anthropic account.
- `/release-notes` — interactive version picker for changelog.
- `/keybindings` — open keyboard shortcuts file.
- `/heapdump` — JS heap snapshot for high-memory debugging.
- `/exit` (alias `/quit`) — exit the CLI. In background session, this detaches; session keeps running.
- `/install-github-app`, `/install-slack-app` — integration setup.
- `/setup-bedrock`, `/setup-vertex` — cloud provider config. Only visible when `CLAUDE_CODE_USE_BEDROCK=1` / `CLAUDE_CODE_USE_VERTEX=1`.
- `/chrome` — Claude in Chrome settings.
- `/ide` — IDE integration status.
- `/terminal-setup` — Shift+Enter and other shortcuts. Only visible in terminals that need it (VS Code, Cursor, etc.).
- `/web-setup` — connect GitHub to Claude Code on the web using local `gh` credentials.
- `/remote-env` — default environment for cloud agents.
- `/passes` — share a free week with friends (eligibility-dependent).
- `/privacy-settings`, `/upgrade`, `/usage-credits` — account.

### Advanced workflows

- `/autofix-pr [prompt]` — spawn a Claude Code on the web session that watches a PR and pushes fixes when CI fails or reviewers comment. Needs `gh` CLI + Claude Code on the web access.
- `/ultraplan <prompt>` — draft a plan in an ultraplan session, review in browser, then execute remotely or send back to terminal.
- `/ultrareview [PR]` — deep multi-agent cloud review. Now preferred as `/code-review ultra`; `/ultrareview` is the alias. 3 free runs on Pro/Max then needs usage credits.
- `/deep-research <question>` **(Workflow)** — fan out web searches, cross-check sources, synthesize a cited report.
- `/workflows` — open the workflow progress view.
- `/schedule [description]` (alias `/routines`) — create / update / list / run routines on Anthropic cloud.

### Bundled skills (full list)

All marked **Skill** above. Recap:

| Skill | Phase |
| --- | --- |
| `/batch` | Large parallel changes |
| `/code-review`, `/simplify`, `/review` | Before ship |
| `/debug`, `/fewer-permission-prompts` | Diagnostics |
| `/loop` | Polling / recurring |
| `/claude-api` | API reference + migration |
| `/run`, `/verify`, `/run-skill-generator` (v2.1.145+) | App-verification |

`/run-skill-generator` is the standout — it generates a per-project skill at `.claude/skills/run-<name>/` capturing how to build + launch your app. Run once per project; re-run when build process changes.

## MCP prompts as commands

MCP servers can expose prompts as commands using the format:

```
/mcp__<server>__<prompt>
```

Discovered dynamically from connected servers. Covered in MCP docs (deferred).

## Cross-OS notes

Most commands are OS-neutral. The few exceptions:

- **`/desktop`** — macOS/Windows only.
- **`/sandbox`** — supported platforms only (macOS Seatbelt, Linux/WSL2 bubblewrap; not native Windows).
- **`/setup-bedrock`, `/setup-vertex`** — environment-gated.
- **`/heapdump`** — writes to `~/Desktop` on Mac/Windows, `~` on Linux without a Desktop folder.

## Practical applications for this repo

Useful default workflow patterns to internalize:

- **Start of every project**: `/init`, then `/memory` to refine the generated `CLAUDE.md`. Once we have client projects, `/mcp` and `/permissions` next.
- **Before any non-trivial task**: `/plan`. Pairs with auto mode (set in `~/.claude/settings.json`, see `../config/permissions.md`).
- **Mid-task context check**: `/context` — much better than guessing. See `../tools/context-window.md` for the underlying budget.
- **Before committing**: `/diff` for visual review, then `/code-review` if the diff is non-trivial.
- **Stuck**: `/rewind` to step back. Cheaper than `/compact` (preserves cache — see `../tools/prompt-caching.md`).
- **End of long session**: `/recap` for a one-line summary you can paste into a PR description or work log.

For the Windows VM workflow: `/desktop` if you adopt the Desktop app, otherwise everything else works the same. The few macOS/Windows-only commands are clearly labeled at runtime.
