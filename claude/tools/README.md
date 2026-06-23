# tools/

How Claude Code's engine works under the hood — the built-in tools and capabilities Claude can invoke, plus the context-budget and caching behavior that shapes how those invocations spend tokens.

Pair with [`claude/config/`](../config/) for the **what's allowed** half (memory + settings + permissions). This dir covers **what's available + how it costs.**

## Current docs

- **[`context-window.md`](context-window.md)** — what fills the 200K token budget at startup and during a session, what survives `/compact`, and the levers for reducing context use (specific prompts, subagents, path-scoped rules, `disable-model-invocation` skills).
- **[`prompt-caching.md`](prompt-caching.md)** — why some session changes are followed by a slow uncached turn, why mid-session `CLAUDE.md` edits don't apply, and how to measure cache hit rate. Includes the per-machine + per-directory scope rule (Mac and Windows VM have separate caches).
- **[`hooks.md`](hooks.md)** — the 28 lifecycle events, five handler types (command / HTTP / MCP tool / prompt / agent), matcher syntax, exit-code-vs-JSON output protocol, `additionalContext` injection, per-event decision control, async hooks, and Windows shell specifics.

## Deferred to later issues

- **`tools-reference.md`** — each built-in tool (Bash, Read, Edit, Write, Grep, Glob, etc.) with parameter notes and output-size limits (`BASH_MAX_OUTPUT_LENGTH`, etc.).
- **`subagents.md`** — built-in subagents, custom subagent files, when to delegate.
- **`skills.md`** — `SKILL.md` format, discovery, `disable-model-invocation`, plugin skills.

## Why these live together

Tools, hooks, skills, and subagents all consume or protect context. Reading [`context-window.md`](context-window.md) makes choices in the other docs easier — e.g. why `disable-model-invocation: true` matters for skills with side effects, why a subagent is a context-saver, why a `PreToolUse` hook is the right enforcement layer instead of a `CLAUDE.md` rule.

## Cross-OS reminder

Most of what's here is OS-neutral. The few exceptions are flagged inline:

- Prompt cache is **per machine + per directory** — Mac and Windows VM never share a cache.
- Native Windows lacks the Bash sandbox (Seatbelt on macOS, bubblewrap on Linux/WSL2 — see `../config/permissions.md`).
- Windows path normalization in tool rules (`C:\Users\...` → `/c/Users/...`).
