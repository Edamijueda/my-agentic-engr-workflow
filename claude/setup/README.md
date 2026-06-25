# setup/

How to prepare a fresh machine to run Claude Code — install, authenticate, and (optionally) wire it into VS Code.

## Current docs

- **[`install.md`](install.md)** — install methods per OS (native installer, Homebrew, WinGet, apt/dnf/apk, npm), update channels + auto-update behavior, version pinning, binary integrity verification, uninstall. **Most directly serves the "prepare a new machine" goal.**
- **[`authentication.md`](authentication.md)** — login flow + 4 account types (Pro/Max, Teams/Enterprise, Console, Bedrock/Vertex/Foundry), credential storage per OS, the 6-tier authentication precedence, `apiKeyHelper`, long-lived OAuth token for CI (`claude setup-token`).
- **[`vs-code.md`](vs-code.md)** — VS Code extension (also works in Cursor + VS Code forks): install, sign-in, prompt-box features, the built-in `ide` MCP server (selection injection — security-relevant), extension settings, shortcuts, URI handler for deep linking, cross-OS notes.

## Suggested first-run order

1. **`install.md`** — get `claude` on the machine. Verify with `claude --version` and `claude doctor`.
2. **`authentication.md`** — `claude` once, follow the browser prompts. Check `/status` to confirm the right auth method is active.
3. **`vs-code.md`** — install the VS Code extension if you use VS Code (or Cursor). **Also install the standalone CLI** so `claude` works in the integrated terminal — the extension bundles a private CLI for the chat panel that doesn't add to PATH.

## What's NOT here

- **JetBrains plugin** — intentionally skipped. This user uses VS Code only.
- **Cloud provider deep dives** (Bedrock / Vertex / Foundry setup steps) — covered in their respective Anthropic docs; `authentication.md` documents the precedence and env vars, not the cloud-side setup.
- **MCP server install** — touched on in `vs-code.md`; full MCP doc deferred to its own issue.

## Cross-OS install matrix (quick reference)

| OS | Recommended | Auto-updates? | Cross-OS quirk |
| --- | --- | --- | --- |
| **macOS** | Native installer (`curl ... install.sh \| bash`) | Yes | Credentials in encrypted Keychain. Easiest setup. |
| **Windows native** | Native installer (PowerShell: `irm ... \| iex`) | Yes | **No sandboxing.** Git for Windows optional (enables Bash tool). Without it, PowerShell tool is used. |
| **WSL 2** | Native installer inside the WSL terminal | Yes | **Sandboxing supported.** Use this if you need sandboxed Bash on Windows. |
| **Linux** | Native installer or apt/dnf/apk repos (signed) | Native yes; package mgrs no | Package mgrs use system upgrade workflow. |

## Day-1 settings worth adopting

In `~/.claude/settings.json`:

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "autoUpdatesChannel": "stable",
  "permissions": {
    "defaultMode": "plan"
  }
}
```

- `"autoUpdatesChannel": "stable"` — ~1 week behind latest, skips regression releases.
- `"permissions.defaultMode": "plan"` — start every session in plan mode. Pairs with the "code captain" goal of seeing the plan before edits happen. (`auto` mode is ignored if set in `.claude/settings.json` or `.claude/settings.local.json` for security — see `../config/permissions.md`.)

VS Code-specific (`claudeCode.initialPermissionMode: "plan"`) is the equivalent inside the extension.
