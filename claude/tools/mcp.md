---
verified_against: 2.1.308
verified_on: 2026-10-30
sources:
  - https://code.claude.com/docs/en/mcp
  - https://code.claude.com/docs/en/mcp-quickstart
---

# MCP — connecting an agent to external systems

[MCP](https://modelcontextprotocol.io/introduction) is an open standard, not a Claude invention — the same servers work with other agents that speak it. That makes it the most portable part of the extension surface.

Connect a server when you notice yourself **copying data into the chat** from somewhere else. That's the signal. Until then, don't.

## Scopes — where a server is configured

| Scope | Flag | Loads in | Stored in |
| --- | --- | --- | --- |
| **Local** (default) | `--scope local` | This project, only for you | `~/.claude.json`, keyed by project path |
| **Project** | `--scope project` | This project, shared via git | `.mcp.json` at the repo root |
| **User** | `--scope user` | All your projects | `~/.claude.json` |

Precedence: local → project → user → plugin-provided → claude.ai connectors. Values are **not** merged across scopes; the first match wins outright.

Project-scoped servers need your approval in an interactive session before they're used — a cloned repo can't approve its own. `claude mcp reset-project-choices` clears those answers.

## Adding one

```bash
# HTTP (recommended for cloud services)
claude mcp add --transport http notion https://mcp.notion.com/mcp

# with a header
claude mcp add --transport http secure-api https://api.example.com/mcp \
  --header "Authorization: Bearer your-token"

# stdio — a local process. Everything after `--` goes to the server.
claude mcp add --env AIRTABLE_API_KEY=KEY --transport stdio airtable \
  -- npx -y airtable-mcp-server

# raw JSON
claude mcp add-json my-server '{"type":"http","url":"https://mcp.example.com/mcp"}'
```

Managing: `claude mcp list`, `claude mcp get <name>`, `claude mcp remove <name>`. In-session, `/mcp` shows status and handles sign-in; `/mcp reconnect all` retries failures.

Transports: **HTTP** is the one to use (`streamable-http` is an accepted alias). **SSE** is deprecated — current versions try HTTP first and fall back. **stdio** runs locally. A JSON entry with a `url` but no `type` is a config error, because it gets read as stdio.

`.mcp.json`, committed, is how a team shares servers:

```json
{
  "mcpServers": {
    "shared-server": { "type": "http", "url": "https://example.com/mcp" }
  }
}
```

## Authentication

Claude Code flags a server as needing auth on a `401`/`403`. Run `/mcp` and follow the browser flow, or `claude mcp login <name>`.

- `${VAR}` and `${VAR:-default}` expand in `command`, `args`, `env`, `url` and `headers`. Credential-looking variables (`ANTHROPIC_API_KEY` and similar) deliberately read as **empty** in a remote `url`/`headers`, so a project config can't exfiltrate them.
- `headersHelper` runs a script at connection time that prints a JSON object of headers. 10-second timeout. A helper supplied by a repo or plugin runs **without** your credential variables.
- `claude mcp logout <name>` before `claude mcp remove <name>` — removing a server deletes the local token but doesn't revoke it at the provider.

## The context cost, and why it's usually fine

This is the part that decides whether connecting a server is cheap.

**Tool search is on by default**, and it defers tool definitions. Only tool **names** and the server's instructions load at session start; full schemas load on demand when Claude needs them. So adding another server has minimal effect on your context window, and there's no fixed cap on how many you can connect — the practical limit is your own budget.

That default is why the honest answer to "what does a server cost per turn?" is *not much* — provided it stays deferred. It doesn't stay deferred when:

- The model doesn't support tool search (Opus 4 / Sonnet 4 / Haiku 4.5 and later do).
- `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` is set, which forces it off and **can't** be overridden by `ENABLE_TOOL_SEARCH`.
- A tool or server is marked `alwaysLoad`.

When tools load upfront instead, adding or removing a server mid-session invalidates the prompt cache — see [`prompt-caching.md`](prompt-caching.md). Deferred, it doesn't.

Server authors: with tool search on, the server-instructions field does the work a skill description does — it tells Claude when to go looking. Descriptions are capped at 4,096 characters upfront and 16,384 when loaded through tool search, so put the searchable words near the start.

## Output limits

| Limit | Default | Override |
| --- | --- | --- |
| Warning threshold | 10,000 tokens | — |
| Max output | 25,000 tokens | `MAX_MCP_OUTPUT_TOKENS` |
| Persist-to-disk | 50,000 characters | per-tool `anthropic/maxResultSizeChars` |
| Tool-call timeout | — | `MCP_TOOL_TIMEOUT`, or `"timeout"` ms per server |

Over the limit, a result is written to a file and replaced in the conversation with the path, so Claude reads it only if it needs to.

## Permissions

Tools are named `mcp__<server>__<tool>`; plugin-bundled ones `mcp__plugin_<plugin>_<server>__<tool>`. Use those names in permission rules, subagent `tools`, skill `allowed-tools` and hook matchers. See [`../config/permissions.md`](../config/permissions.md).

## Security

A server that fetches external content is a **prompt-injection surface** — content it returns is data, not instructions. Connect servers you trust, and treat the output of untrusted ones accordingly.

- `disabledMcpjsonServers` in any settings file blocks a `.mcp.json` server in every permission mode.
- `--strict-mcp-config` limits a session to the servers passed with `--mcp-config`.
- Whatever a server can reach, it reaches as you.

## See also

- [`../../shared/principles/defer-until-friction.md`](../../shared/principles/defer-until-friction.md) — the copying-data-into-chat signal is the friction; wait for it.
- [`tools-reference.md`](tools-reference.md) — `ToolSearch` and the built-in tool roster.
- [`plugins.md`](plugins.md) — plugins can bundle MCP servers.
