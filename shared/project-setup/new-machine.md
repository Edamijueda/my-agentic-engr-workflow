# Setting up a new machine — checklist

Once per machine. Companion to [`new-project.md`](new-project.md), which runs once per project.

After this checklist, you can `cd` into any project and run `claude` productively. Personal preferences are in place, credentials work, IDE is wired up.

## TL;DR — five steps

1. **Install** the CLI (native installer auto-updates).
2. **Authenticate** with `claude` (browser-based OAuth).
3. **Drop a personal `~/.claude/settings.json`** with plan-mode default + channel preference.
4. **Install the VS Code extension** (if you use VS Code/Cursor) + the standalone CLI (the extension's bundled CLI doesn't add to PATH).
5. **Verify** with `claude doctor`, `/status`, and a smoke-test session.

The full mechanics live in [`claude/setup/install.md`](../../claude/setup/install.md), [`claude/setup/authentication.md`](../../claude/setup/authentication.md), [`claude/setup/vs-code.md`](../../claude/setup/vs-code.md). This doc is the literal sequence.

---

## Step 1: Pick the install method

| OS | Recommended | Why |
| --- | --- | --- |
| **macOS** | Native installer (`curl`) | Auto-updates, simplest path. |
| **Windows native** | Native installer (PowerShell `irm`) | Auto-updates. **No sandboxing.** Git for Windows optional (enables Bash tool). |
| **Windows + WSL 2** | Native installer inside WSL | Sandboxing supported. Use if you need sandboxed Bash. |
| **Linux** | Native installer or apt/dnf/apk | Native auto-updates; package manager uses system upgrade. |

Full decision tree + Homebrew / WinGet / npm alternatives: [`claude/setup/install.md`](../../claude/setup/install.md).

### Quick install commands

**macOS / Linux / WSL:**
```bash
curl -fsSL https://claude.ai/install.sh | bash
```

**Windows PowerShell:**
```powershell
irm https://claude.ai/install.ps1 | iex
```

**Windows CMD:**
```batch
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

> **Identifying your Windows shell:** prompt shows `PS C:\` in PowerShell, `C:\` (no `PS`) in CMD. `'irm' is not recognized` = you're in CMD. `The token '&&' is not a valid statement separator` = you're in PowerShell.

### Verify install

```bash
claude --version       # should print a version
claude doctor          # detailed health check
```

If `claude` is not found, see [`claude/setup/install.md`](../../claude/setup/install.md) → "Verify install."

---

## Step 2: Authenticate

```bash
claude
```

First launch opens a browser for OAuth. Three fallbacks:

| Symptom | Fix |
| --- | --- |
| Browser doesn't open | Press `c` to copy the URL to clipboard, paste into browser manually. |
| Browser shows a code, doesn't redirect back | Paste the code into the terminal at the prompt. |
| You're in WSL / SSH / a container | Expect the code-paste flow — Claude Code's local callback server isn't reachable. |

After authentication, run `/status` to confirm. The active auth method should match the plan you expect (Pro / Max / Teams / Enterprise / Console).

> **Free Claude.ai accounts don't include Claude Code access.** You need a paid plan or a Console account.

Full auth flow incl. Bedrock/Vertex/Foundry, `apiKeyHelper`, and the Pro+API-key gotcha: [`claude/setup/authentication.md`](../../claude/setup/authentication.md).

---

## Step 3: Personal day-1 settings (`~/.claude/settings.json`)

This is the file that follows YOU across all your projects on this machine. **Doesn't sync to your other machines** — write it on each one.

Drop this in:

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "autoUpdatesChannel": "stable",
  "permissions": {
    "defaultMode": "plan"
  }
}
```

Why each line:
- **`autoUpdatesChannel: "stable"`** — ~1 week behind latest, skips regression releases. Lower-friction default.
- **`permissions.defaultMode: "plan"`** — plan mode by default for every session. **Must be in user scope** — `auto` is intentionally ignored from project/local settings as a repo-spoof guard, and the same logic argues for keeping plan-as-default personal. See [`shared/principles/enforcement-vs-influence.md`](../principles/enforcement-vs-influence.md).

The `$schema` line enables autocomplete + inline validation in any editor that understands JSON schema (VS Code, JetBrains, Cursor).

Optional additions:

```json
{
  "theme": "dark",
  "editorMode": "vim",
  "tui": "fullscreen",
  "preferredNotifChannel": "terminal_bell"
}
```

For the full settings reference: [`claude/config/settings.md`](../../claude/config/settings.md).

---

## Step 4: Personal user-scope `CLAUDE.md` (optional)

`~/.claude/CLAUDE.md` is loaded into every session you run on this machine, alongside any project `CLAUDE.md`.

**When to write one:**
- You have rules that apply to your work regardless of project (style preferences, "always run tests before claiming done," etc.).
- You want certain behaviors enforced across both work and side projects without per-project setup.

**When NOT to write one:**
- The rule belongs to a specific project — put it in `./CLAUDE.md` there instead.
- You haven't actually noticed the friction yet. **Defer until friction.** See [`shared/principles/defer-until-friction.md`](../principles/defer-until-friction.md).

Keep it tight. Same 200-line guideline as project memory.

Full mechanics: [`claude/config/memory.md`](../../claude/config/memory.md).

---

## Step 5: VS Code extension (if you use VS Code/Cursor)

If you use VS Code, Cursor, or a VS Code fork (Devin Desktop, Kiro), install the extension:

| Path | How |
| --- | --- |
| Direct URI (VS Code) | `vscode:extension/anthropic.claude-code` |
| Direct URI (Cursor) | `cursor:extension/anthropic.claude-code` |
| Extensions view | `Cmd+Shift+X` / `Ctrl+Shift+X` → search "Claude Code" |

> **Gotcha:** the extension bundles a private CLI for the chat panel — **it does not add `claude` to your PATH.** You also need the standalone CLI install (Step 1) for `claude` to work in the integrated terminal. Conversation history is shared between the chat panel and the standalone CLI.

### The env-inheritance gotcha

If you have env vars set in your shell (e.g. `ANTHROPIC_API_KEY` or `CLAUDE_CODE_USE_BEDROCK`), launch VS Code from a terminal:

```bash
code .
```

Otherwise VS Code launched from the Dock/Start menu doesn't inherit your shell env, and you'll see a sign-in prompt instead of the configured auth path.

Optional extension settings worth knowing:

| Setting | Recommended | Why |
| --- | --- | --- |
| `claudeCode.initialPermissionMode` | `"plan"` | Matches your CLI plan-mode default. |
| `claudeCode.autosave` | `true` (default) | Saves before Claude reads or writes — prevents stale-on-disk surprises. |
| `claudeCode.respectGitIgnore` | `true` (default) | Keeps gitignored files out of `@`-mentions. |
| `claudeCode.useTerminal` | `false` (default) | Graphical panel for chat; use integrated terminal for CLI access. |

Full extension reference incl. the built-in `ide` MCP server (security-relevant: it auto-sends your current selection + active file path to Claude on every prompt): [`claude/setup/vs-code.md`](../../claude/setup/vs-code.md).

---

## Step 6: Final verification

In any project directory:

```bash
cd ~/some-project
claude
```

Inside the session:

```
/status       # confirms account + model + connectivity
/doctor       # diagnoses any setup issues
/context      # shows what's in context at startup
```

Quick smoke test prompt:

```
What's the current branch and recent changes here?
```

The agent should answer using its environment + git context block. If it can't, the install or auth has an issue — start with `/doctor` output.

---

## Cross-machine reality

What automatically follows you when you switch from Mac to Windows VM (or any other machine):

| Item | Syncs across machines? | Lives where |
| --- | --- | --- |
| **Project `CLAUDE.md`** | ✅ if committed | repo |
| **`.claude/settings.json`** | ✅ if committed | repo |
| **`.claude/rules/`, `.claude/agents/`, `.claude/skills/`** | ✅ if committed | repo |
| **Personal `~/.claude/CLAUDE.md`** | ❌ | local home dir |
| **Personal `~/.claude/settings.json`** | ❌ | local home dir |
| **Auto memory** (`~/.claude/projects/<project>/memory/`) | ❌ machine-local | local home dir |
| **Credentials** | ❌ re-auth on each machine | macOS Keychain / `~/.claude/.credentials.json` |
| **Prompt cache** | ❌ per machine + per directory | server-side |

**Practical**: anything you commit to the repo follows you. Anything in `~/.claude/` does not. Run this `new-machine.md` checklist on each machine to recreate the personal layer.

If you want personal `~/.claude/` to follow you, manage it via a separate private dotfiles-style repo and `git pull` on each machine. The public `my-agentic-engr-workflow` repo isn't the right place.

For the durability + sync axes in detail: [`shared/principles/durability-ladder.md`](../principles/durability-ladder.md).

---

## Optional: CI machines

For machines that can't do interactive browser login (CI runners, build servers, headless containers):

```bash
claude setup-token
```

Walks you through OAuth once and prints a **one-year token**. The command doesn't save it — copy and store:

```bash
export CLAUDE_CODE_OAUTH_TOKEN=your-token
```

Requires Pro/Max/Team/Enterprise. Inference-only — cannot establish Remote Control sessions. `--bare` mode doesn't read this env var; use `ANTHROPIC_API_KEY` for bare mode.

Calendar a renewal before the year is up.

---

## Cross-OS specifics

### macOS

- Credentials stored in Keychain — no manual file.
- Sandbox via Seatbelt.
- Default install: native `curl | bash`. Auto-updates.

### Windows native

- Credentials in `%USERPROFILE%\.claude\.credentials.json` (user-profile ACLs).
- **No sandbox** — compensate with stricter `permissions.deny` rules at the project level.
- Bash tool requires [Git for Windows](https://git-scm.com/downloads/win). Without it, Claude Code falls back to the PowerShell tool.
- If Git Bash is installed but Claude Code can't find it, set in `~/.claude/settings.json`:
  ```json
  { "env": { "CLAUDE_CODE_GIT_BASH_PATH": "C:\\Program Files\\Git\\bin\\bash.exe" } }
  ```

### Windows + WSL 2

- Sandbox supported (bubblewrap).
- Install the **Linux** installer inside the WSL terminal (`curl | bash`), not from PowerShell or CMD.
- Auto-update works normally.

### Linux

- Credentials in `~/.claude/.credentials.json` (mode `0600`).
- Sandbox via bubblewrap.
- Package managers (apt/dnf/apk) available with signed repos but no auto-update.

---

## See also

- [`new-project.md`](new-project.md) — once per project (memory + permissions + plan-mode default).
- [`claude/setup/install.md`](../../claude/setup/install.md) — full install reference (channels, version pinning, binary integrity, uninstall).
- [`claude/setup/authentication.md`](../../claude/setup/authentication.md) — full auth reference (6-tier precedence, `apiKeyHelper`, the Pro+API-key gotcha).
- [`claude/setup/vs-code.md`](../../claude/setup/vs-code.md) — VS Code extension in detail (built-in MCP server, settings, URI handler).
- [`shared/principles/durability-ladder.md`](../principles/durability-ladder.md) — what crosses machines, what doesn't.
