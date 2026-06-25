# VS Code extension

The recommended way to use Claude Code in VS Code (and Cursor, Devin Desktop, Kiro, and other forks). Distilled from `https://code.claude.com/docs/en/vs-code`.

## Mental model

**Two surfaces share one conversation history**:

| | Extension chat panel | Integrated terminal |
| --- | --- | --- |
| Uses CLI? | **Yes, but bundled-private copy** | Standalone CLI (you install it) |
| Adds `claude` to PATH? | No | Yes (via standalone install) |
| All built-in commands? | Subset (type `/` to see) | Yes |
| `!` bash shortcut, tab completion | No | Yes |
| MCP management | Add via CLI, manage with `/mcp` | Full |
| Checkpoints | Yes | Yes |

> The extension bundles its own CLI for the chat panel. To run `claude` in VS Code's integrated terminal, you also need the **standalone CLI install** (see `install.md`). Conversation history is shared — switch between surfaces freely.

## Prerequisites

- **VS Code 1.98.0+**
- Any paid Claude account (Pro/Max/Team/Enterprise) **or** Claude Console account. **No API key required.**
- Cloud-provider users (Bedrock/Vertex/Foundry): need to disable the login prompt — see "Third-party providers" below.

## Install

| Method | Source |
| --- | --- |
| **VS Code direct URI** | `vscode:extension/anthropic.claude-code` |
| **Cursor direct URI** | `cursor:extension/anthropic.claude-code` |
| **Extensions view** | `Cmd+Shift+X` (Mac) / `Ctrl+Shift+X` (Win/Linux) → search "Claude Code" |
| **VS Code forks** (Devin Desktop, Kiro) | Same Extensions view search, or Open VSX registry |

If the extension doesn't appear after installing: restart, or `Developer: Reload Window` from Command Palette.

## Opening the Claude panel

Four entry points:

| Where | Visible when |
| --- | --- |
| **Spark icon in Editor Toolbar** (top-right of editor) | A file is open |
| **Spark icon in Activity Bar** (left sidebar) | Always |
| **Command Palette** → "Claude Code" → "Open in New Tab" / etc. | Always |
| **`✱ Claude Code` in Status Bar** (bottom-right) | Always — works without a file open |

The panel is **draggable** — secondary sidebar (right), primary sidebar (left), or as an editor tab. Claude remembers your preferred location.

## First sign-in + the env gotcha

First open shows a sign-in screen. Click **Sign in** and complete in browser.

If you see **"Not logged in · Please run /login"** later, the sign-in screen reopens automatically. If not: `Developer: Reload Window`.

> **If `ANTHROPIC_API_KEY` is set in your shell but you still see the sign-in prompt** → VS Code didn't inherit your shell environment. Launch VS Code from a terminal with `code .` so it inherits your env. Alternatively, sign in with your Claude account directly. (This is the same shell-env-inheritance trap from `authentication.md`.)

After sign-in, a **Learn Claude Code** checklist appears. Dismiss with X or `Extensions → Claude Code → Hide Onboarding` to suppress.

## Prompt box

### Selection awareness

Claude automatically sees your selected text — footer shows line count. Click the selection indicator to toggle (eye-slash = hidden from Claude).

`Option+K` / `Alt+K` inserts an @-mention with the file path + line range (e.g. `@app.ts#5-10`).

### @-mentions

| Pattern | Effect |
| --- | --- |
| `@auth` | Fuzzy match — finds `auth.js`, `AuthService.ts`, etc. |
| `@src/components/` | **Trailing slash** = folder |
| `@app.ts#5-10` | Specific line range (added via `Option+K`/`Alt+K`) |
| `@terminal:name` | Terminal output by terminal title |
| `Shift+drag` files into the box | Add as attachments |

For large PDFs, ask for specific pages — single, range, or open-ended.

### Permission modes

Click the mode indicator at the bottom of the prompt box:

| Mode | Behavior |
| --- | --- |
| Normal | Asks before each action |
| **Plan** | Describes what it will do; **opens plan as a full markdown document** where you add inline comments before Claude begins |
| Accept edits | Auto-accepts file edits |

Set the default via `claudeCode.initialPermissionMode` (`default`, `plan`, `acceptEdits`, `bypassPermissions`). Same modes as the CLI (see `../config/permissions.md`).

### Command menu (`/`)

Click `/` or type it. Provides:
- Attach files, switch models, toggle extended thinking.
- `/usage`, `/remote-control`.
- **Customize section**: MCP servers, hooks, memory, permissions, plugins.
- Items with terminal icons open in the integrated terminal.

### Other niceties

- **Extended thinking** — toggle via `/`. Reasoning shows as collapsed blocks; `Ctrl+O` expands or collapses all in the session.
- **`Shift+Enter`** for newline without sending. Also works in the "Other" input of question dialogs.
- **Context indicator** — shows how much of the context window you're using. `/compact` to free space.

## Session history

Top of the Claude panel. Search by keyword, browse by time (Today, Yesterday, Last 7 days, …). New sessions get AI-generated titles from your first message. Hover to rename or remove.

### Resume cloud sessions from claude.ai

If you use Claude Code on the web, those cloud sessions appear under the **Remote** tab in session history. Requires **Claude.ai Subscription auth** (not Console).

> Only web sessions started with a GitHub repository appear. Resuming downloads the conversation locally — **changes are not synced back to claude.ai.**

## `/usage` dialog (v2.1.174+)

Account & usage panel:
- Plan name, signed-in account.
- Usage bars: current session + week. Time to limit reset.
- **Flags behaviors that account for 10%+ of recent usage** — cache misses, long context, subagent-heavy or highly parallel sessions, each with a tip.
- **Attribution tables**: usage per skill, subagent, plugin, MCP server.
- Day/Week toggle (last 24h vs 7d).

**Local-only**: computed from sessions on this machine. Won't see usage from your Windows VM or claude.ai sessions.

## Multiple conversations

`Open in New Tab` (`Cmd+Shift+Esc` / `Ctrl+Shift+Esc`) or `Open in New Window` from Command Palette. Each conversation has its own history and context.

Spark icon dot indicators:
- **Blue dot** = permission request pending.
- **Orange dot** = Claude finished while tab was hidden.

## Switching between extension and CLI

Same conversation history. To continue an extension conversation in the CLI:

```bash
claude --resume
```

Opens an interactive picker.

To connect Claude Code running in an **external** terminal back to VS Code: `/ide` inside Claude Code.

## Checkpoints (rewind)

Hover any message → rewind button → three options:
- **Fork conversation from here** — new conversation branch, code changes intact.
- **Rewind code to here** — revert file changes, keep full conversation history.
- **Fork conversation and rewind code** — both.

## MCP management

Add servers via the integrated terminal:

```bash
claude mcp add --transport http github https://api.githubcopilot.com/mcp/ \
  --header "Authorization: Bearer YOUR_GITHUB_PAT"
```

Manage existing servers with `/mcp` in the chat panel — enable/disable, reconnect, OAuth.

> Settings configured via the extension are also available in the CLI, and vice versa — both sides use the same files.

## Git worktrees

```bash
claude --worktree feature-auth   # or -w
```

Spawn an isolated worktree with its own files and branch. Prevents Claude instances from interfering when working on different tasks.

## Third-party providers (Bedrock / Vertex / Foundry)

1. Enable **Disable Login Prompt** in extension settings (or `claudeCode.disableLoginPrompt: true`).
2. Configure the provider in `~/.claude/settings.json` (shared between extension and CLI) — see the provider's setup doc.

## The built-in IDE MCP server (security-relevant)

When the extension is active, it runs a **local MCP server named `ide`** that the CLI connects to automatically. Powers diff viewer, `@`-mentions, Jupyter cell execution.

**Hidden from `/mcp`** because there's nothing to configure. But you need to know it exists if you write `PreToolUse` hooks that allowlist MCP tools.

### Selection + open-file context

While connected, the CLI **includes your current editor selection and the active file's path as context on every prompt you send**. Transcript shows `⧉ Selected N lines from <file>`.

> To **exclude a sensitive file** (`.env`, secrets), add a `Read(...)` deny rule for its path. A matching deny rule prevents both the selected text AND the open-file notice from reaching Claude.

### Transport + auth

- Binds to `127.0.0.1` on a random high port — not reachable from other machines.
- Each activation generates a fresh random auth token in a lock file under `~/.claude/ide/`.
- Lock file: `0600` permissions inside a `0700` directory — only the user running VS Code can read it.

### Tools the server hosts

A dozen tools, but only two visible to Claude:

| Tool | Effect | Writes? |
| --- | --- | --- |
| `mcp__ide__getDiagnostics` | Language-server diagnostics (Problems panel content). Optionally scoped to one file. | No |
| `mcp__ide__executeCode` | Runs Python in active Jupyter notebook's kernel. **Always prompts via Quick Pick** (Execute / Cancel / `Esc` = cancel). Refuses if no active notebook, Jupyter ext (`ms-toolsai.jupyter`) not installed, or kernel isn't Python. | Yes |

> **Quick Pick confirmation is separate from `PreToolUse` hooks.** An allowlist entry for `mcp__ide__executeCode` lets Claude **propose** running a cell; the Quick Pick is what lets it **actually** run.

The other ten tools are internal RPC (open diffs, read selections, save files) and filtered out before the tool list reaches Claude.

## Extension settings — the useful ones

Open via `Cmd+,` / `Ctrl+,` → Extensions → Claude Code, or type `/` → **General Config**.

| Setting | Default | Notes |
| --- | --- | --- |
| `useTerminal` | `false` | Launch in terminal mode instead of graphical panel. |
| `initialPermissionMode` | `default` | `default`, `plan`, `acceptEdits`, `bypassPermissions`. Doesn't accept `auto` — set in `~/.claude/settings.json` instead. |
| `preferredLocation` | `panel` | `sidebar` (right) or `panel` (tab). |
| `autosave` | `true` | Auto-save files before Claude reads/writes. |
| `useCtrlEnterToSend` | `false` | Use `Ctrl/Cmd+Enter` instead of `Enter` to send. |
| `enableNewConversationShortcut` | `false` | Enable `Cmd/Ctrl+N` to start a new conversation. Requires Claude focused. |
| `enableReopenClosedSessionShortcut` | `true` | `Cmd/Ctrl+Shift+T` reopens the most recently closed Claude session. Falls through to VS Code default if last closed wasn't Claude. |
| `respectGitIgnore` | `true` | Exclude `.gitignore` patterns from file searches. |
| `usePythonEnvironment` | `true` | Activate workspace's Python environment. Requires VS Code Python extension. |
| `environmentVariables` | `[]` | Env vars for the Claude process. **Prefer `~/.claude/settings.json` `env` for shared config.** |
| `disableLoginPrompt` | `false` | For third-party providers. |
| `allowDangerouslySkipPermissions` | `false` | Adds Bypass permissions to mode selector. **Sandboxes only.** |
| `claudeProcessWrapper` | — | Executable to launch the Claude process. Set this to point at a separately installed `claude` if the bundled build isn't available for your platform. |

> Add `"$schema": "https://json.schemastore.org/claude-code-settings.json"` to `~/.claude/settings.json` for autocomplete + inline validation in VS Code's JSON editor.

## VS Code commands + shortcuts

Some shortcuts depend on whether the editor or Claude's prompt box has focus. `Cmd+Esc` / `Ctrl+Esc` toggles.

| Command | Shortcut | Notes |
| --- | --- | --- |
| Focus Input | `Cmd+Esc` / `Ctrl+Esc` | Toggle editor ↔ Claude. |
| Open in New Tab | `Cmd+Shift+Esc` / `Ctrl+Shift+Esc` | New conversation as editor tab. |
| New Conversation | `Cmd+N` / `Ctrl+N` | Requires Claude focused **and** `enableNewConversationShortcut: true`. |
| Reopen Closed Session | `Cmd+Shift+T` / `Ctrl+Shift+T` | Falls through to VS Code default when last closed wasn't Claude. |
| Insert @-Mention | `Option+K` / `Alt+K` | Requires editor focused. |
| Open in Side Bar / Terminal / New Window | — | Command Palette only. |
| Show Logs | — | Extension debug logs. |
| Logout | — | Sign out. |

> Command Palette → "Claude Code" lists everything.

## URI handler (deep linking)

`vscode://anthropic.claude-code/open` — opens a Claude Code tab. Useful for shell aliases, bookmarklets, or any script that opens URLs.

Optional query params:
- `prompt` — text to pre-fill (URL-encoded, **not auto-submitted**).
- `session` — session ID to resume. Must belong to the workspace currently open. If not found, starts fresh. If already open, focuses that tab.

Invoke per OS:

```bash
# macOS
open "vscode://anthropic.claude-code/open?prompt=review%20my%20changes"

# Linux
xdg-open "vscode://anthropic.claude-code/open"
```

```powershell
# Windows PowerShell
Start-Process "vscode://anthropic.claude-code/open"
```

```cmd
:: Windows cmd — `start` treats first quoted arg as window title, so pass an empty title first
start "" "vscode://anthropic.claude-code/open"
```

For terminal sessions (not VS Code tabs), use the CLI's `claude-cli://` handler instead.

## Troubleshooting (the common ones)

### Spark icon not visible

Editor Toolbar Spark icon only appears when a file is open. If still missing:
1. Open a file (folder-only isn't enough).
2. VS Code 1.98.0+ (Help → About).
3. `Developer: Reload Window`.
4. Disable other AI extensions (Cline, Continue) temporarily.
5. Check workspace trust — extension **doesn't work in Restricted Mode**.

Alt: Status Bar `✱ Claude Code` (always visible) or Command Palette.

### `Cmd+Esc` does nothing on macOS

On **macOS Tahoe and later**, the system Game Overlay shortcut is bound to `Cmd+Esc` by default and intercepts the keypress before VS Code sees it.

Free the shortcut:
- System Settings → Keyboard → Keyboard Shortcuts → Game Controllers → uncheck Game Overlay.

Or rebind: VS Code Keyboard Shortcuts editor (`Cmd+K Cmd+S`), search "Claude Code: Focus input", assign a different binding.

### Claude Code never responds

1. Check internet.
2. Start a new conversation.
3. Try the CLI (`claude` in integrated terminal) — better error messages.

## Uninstall

Extensions view (`Cmd+Shift+X` / `Ctrl+Shift+X`) → search "Claude Code" → Uninstall.

To wipe extension storage too:

```bash
# macOS
rm -rf ~/Library/Application\ Support/Code/User/globalStorage/anthropic.claude-code

# Linux
rm -rf ~/.config/Code/User/globalStorage/anthropic.claude-code
```

```powershell
# Windows PowerShell
Remove-Item -Recurse -Force "$env:APPDATA\Code\User\globalStorage\anthropic.claude-code"
```

(For full Claude Code uninstall including `~/.claude/`, see `install.md`.)

## Cross-OS notes

- Same extension on macOS and Windows. Settings UI identical.
- **Windows shortcuts**: `Ctrl+...` everywhere. The `Cmd+Esc` Game Overlay issue is macOS-only.
- **Cursor + VS Code forks**: same extension (via Open VSX). Cursor URI: `cursor:extension/anthropic.claude-code`.
- VS Code's "Use Terminal" mode (`claudeCode.useTerminal: true`) effectively gives you the same CLI experience as the standalone install — useful in environments where the panel UI feels too heavy.

## Practical applications for this repo

- **Day-1 settings to consider**:
  - `initialPermissionMode: "plan"` — start in plan mode by default (matches "code captain" goal).
  - `autosave: true` (default) — saves before Claude reads / writes.
  - `respectGitIgnore: true` (default) — keeps gitignored files out of file searches.
  - `useTerminal: false` (default) — graphical panel for chat, integrated terminal for CLI access.
- **Critical**: also install the standalone CLI (see `install.md`) so `claude` works in the integrated terminal. The extension's bundled CLI doesn't add to PATH.
- **Selection-injection awareness**: the built-in IDE MCP server **always** sends your current selection + active file path to Claude on every prompt. For sensitive files, add a `Read(./.env*)`-style deny rule — covers both the selection and the open-file notice.
- **For the Windows VM**: same extension, same install steps. Cursor URI also works if you switch to Cursor on Windows.
- **JetBrains is not used here** — skipping JetBrains plugin docs.
