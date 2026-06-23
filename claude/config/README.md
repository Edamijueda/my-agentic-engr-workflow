# config/

How Claude Code is told what to do (memory) and what it's allowed to do (settings + permissions). These three docs are the foundation — every other config concern (hooks, MCP, plugins, sandboxing) sits on top.

## The three files

- **[`memory.md`](memory.md)** — `CLAUDE.md` (you write) and auto memory (Claude writes). Scope ladder from managed-policy down to `CLAUDE.local.md`. Imports, `.claude/rules/`, `/init`, `/memory`, `/compact` behavior. **This is influence, not enforcement.**
- **[`settings.md`](settings.md)** — `settings.json`: scopes, precedence, hot-reload, the keys most likely to matter for solo coding-agent use. Plus the neighboring files (`~/.claude.json`, `.mcp.json`, managed-settings delivery channels).
- **[`permissions.md`](permissions.md)** — permission rules (allow / ask / deny) and the six permission modes. Tool-specific syntax (Bash, PowerShell, Read/Edit, WebFetch, MCP, Agent, Cd). **This is enforcement.**

## Mental model — when to reach for what

| You want to... | Use |
| --- | --- |
| Tell Claude *how to behave* in a project | `CLAUDE.md` (memory.md) |
| Tell Claude *how to behave* across all your projects | `~/.claude/CLAUDE.md` (memory.md) |
| Configure the tool itself (theme, model, env vars, hooks) | `settings.json` (settings.md) |
| Allow / block specific tools or commands | `permissions` in `settings.json` (permissions.md) |
| Allow / block at the OS level (Bash subprocesses) | `sandbox` in `settings.json` (settings.md) |
| Force something to happen at a specific moment | Hook (covered in a later doc) |
| Make Claude *not be able to even try* something | Permission deny rule or hook, not memory |

The single most useful framing: **memory shapes intent, settings shape configuration, permissions shape what runs.** Memory and settings can be ignored or misread; permissions and hooks cannot.

## Suggested read order

1. **`memory.md`** first — it's the cheapest lever and applies to every session immediately. Most of what you'd type into chat repeatedly belongs here.
2. **`settings.md`** second — once you know what you'd configure, you'll see why the scope system matters.
3. **`permissions.md`** last — built on top of settings, and only worth the depth once you have something to protect.

If you're starting a new client project on a fresh machine, the minimum is:
- A project `CLAUDE.md` with build/test commands and any non-obvious conventions.
- A project `.claude/settings.json` with `permissions.deny` for `.env`, secrets, and high-blast-radius commands.
- Optionally a personal `~/.claude/settings.json` with theme/editor/notification preferences.

## What's NOT here yet

These belong in `config/` eventually but are deferred to later issues:

- **`hooks.md`** — `PreToolUse`, `ConfigChange`, `InstructionsLoaded`, HTTP hooks, etc. The enforcement layer beyond permissions.
- **`mcp.md`** — `.mcp.json`, managed MCP servers, claude.ai connectors. The extension layer for external systems.
- **`plugins.md`** — `enabledPlugins`, marketplaces, the plugin trust model.
- **`sandboxing.md`** — sandbox config in detail (currently a short section in `settings.md`).
- **`statusline.md` and `fileSuggestion.md`** — smaller features mentioned in `settings.md`.

When we draft each of these, lift the corresponding section out of `settings.md` and let `settings.md` link to it instead.

## Cross-OS reminder

Every one of these mechanisms works on both macOS and Windows, but paths differ. Highlights captured inline in each doc:

- Managed-policy paths.
- `~/.claude` → `%USERPROFILE%\.claude` on Windows.
- Sandbox available on macOS (Seatbelt) and Linux/WSL2 (bubblewrap); **not on native Windows** — compensate with stricter permission rules.
- Windows symlinks need elevation; prefer `@AGENTS.md` import to `ln -s`.
- `defaultShell: "powershell"` + `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` if Git for Windows isn't installed.
