# setup/

How to prepare a fresh machine to run Claude Code — install, authenticate, and (optionally) wire it into VS Code. Written for macOS; other platforms link out to the official docs.

## Current docs

- **[`install.md`](install.md)** — install, update channels and auto-update behaviour, version pinning. **Most directly serves the "prepare a new machine" goal.**
- **[`authentication.md`](authentication.md)** — login flow + account types, credential storage, the 7-tier authentication precedence, `apiKeyHelper`, long-lived OAuth token for CI (`claude setup-token`).
- **[`vs-code.md`](vs-code.md)** — VS Code extension (also works in Cursor + VS Code forks): install, sign-in, prompt-box features, the built-in `ide` MCP server (selection injection — security-relevant), extension settings, shortcuts, URI handler for deep linking.

## Suggested first-run order

1. **`install.md`** — get `claude` on the machine. Verify with `claude --version` and `claude doctor`.
2. **`authentication.md`** — `claude` once, follow the browser prompts. Check `/status` to confirm the right auth method is active.
3. **`vs-code.md`** — install the VS Code extension if you use VS Code (or Cursor). **Also install the standalone CLI** so `claude` works in the integrated terminal — the extension bundles a private CLI for the chat panel that doesn't add to PATH.

## What's NOT here

- **JetBrains plugin** — intentionally skipped. VS Code only.
- **Cloud provider deep dives** (Bedrock / Vertex / Foundry setup steps) — covered in their respective Anthropic docs; `authentication.md` documents the precedence and env vars, not the cloud-side setup.
- **MCP server install** — touched on in `vs-code.md`; the full reference is [`../tools/mcp.md`](../tools/mcp.md).

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
