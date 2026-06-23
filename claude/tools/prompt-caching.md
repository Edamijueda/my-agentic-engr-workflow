# Prompt caching: what invalidates the cache, what keeps it

Why some session changes are followed by a slow uncached turn, why mid-session edits to `CLAUDE.md` don't take effect, and how to measure the hit rate. Distilled from `https://code.claude.com/docs/en/prompt-caching`.

## Mental model

Each turn, Claude Code sends a full request: system prompt, project context, every prior message and tool result, plus your new message. The API caches by **matching the prefix (the start of each request)** against content it recently processed. New content is appended at the end, so most of each request is identical to the previous one.

**The match is exact, and there's no per-file or per-segment caching.** A change *anywhere* in the prefix recomputes everything after it.

To keep the prefix stable, Claude Code orders each request from most stable to least:

| Layer | Content | Changes when |
| --- | --- | --- |
| **System prompt** | Core instructions, tool definitions, output style | Tool set changes; Claude Code upgrades |
| **Project context** | `CLAUDE.md`, auto memory, unscoped rules | Session start; `/clear`; `/compact` |
| **Conversation** | Your messages, Claude's responses, tool results | Every turn |

A change to the conversation layer leaves system prompt + project context cached. A change to the system prompt invalidates everything below it.

## The cache key includes more than text

Three knobs that aren't part of the request text but **are part of the cache key** — touch them mid-session and the whole conversation re-reads:

| Knob | What invalidation looks like |
| --- | --- |
| **Model** | Each model has its own cache. `/model` switch = full recompute. |
| **Effort level** | Each effort level has its own cache. `/effort` switch = full recompute. Claude Code shows a confirmation dialog before applying mid-session. |
| **Fast mode** | Toggling fast mode on adds a header that's part of the key. Costs once per conversation. After the first fast-mode turn, toggling fast mode off then back on keeps the cache (v2.1.86+). |

**`opusplan` quirk:** the model setting resolves to Opus during plan mode and Sonnet during execution, so **each plan-mode toggle is a model switch** = full recompute.

## Actions that invalidate the cache

Next turn = slower + more expensive, then the new prefix is cached.

### Switching models
Includes explicit `/model`, the `opusplan` toggle on entering/leaving plan mode, and **automatic Fable 5 fallback to Opus** when a safety classifier flags a request.

### Changing effort level
Same shape as model switch. Confirmation dialog before applying. A change that resolves to the level already in effect (e.g. setting the default explicitly) skips the dialog and keeps the cache.

### Turning on fast mode
Adds a header to the cache key — full re-read. On non-Opus models, enabling fast mode also switches the model, so you eat two invalidations at once. After the first fast-mode turn, the header is sticky: subsequent on/off toggles, the auto fallback to standard speed after a rate limit, and turning it back on later all keep the cache. `/clear` and `/compact` reset this anyway.

### Connecting / disconnecting an MCP server
Depends on whether tool definitions are **deferred** or **loaded into the prefix**:

| MCP tool delivery | Mid-session change |
| --- | --- |
| **Deferred** (default on supported models) | Cache stays — additions append after the cache breakpoint. |
| **In prefix** | Full re-read. |

Tools land in the prefix when: tool search is unavailable (Haiku, Vertex AI, custom `ANTHROPIC_BASE_URL` gateways), the server/tool is marked `alwaysLoad`, or threshold-based loading kept them upfront.

Sneaky cause: a stdio server's process exits, an HTTP session expires, or a server reconnects automatically after a transient failure — counts as a connect/disconnect.

**Editing the MCP config by itself doesn't change the cache** — it takes effect on the next restart.

> Toggling the advisor tool (`/advisor`) is an exception: its definition sits after the cache breakpoint, so enabling/disabling keeps the prefix intact.

### Enabling / disabling a plugin
Depends on what the plugin provides:

| Plugin component | Mid-session effect |
| --- | --- |
| Skills, commands, agents, hooks, LSP, monitors, themes | Cache survives (append-only). |
| MCP servers | Same rules as MCP changes above. |

Changes apply on `/reload-plugins` or a new session — **not** at the moment of `/plugin install / enable / disable`. As of v2.1.163, `/reload-plugins` warns if a reload would trigger a full re-read; pass `--force` to apply anyway.

Disabling a plugin you enabled earlier in the session can hit the **previous prefix's cache** if it's still within TTL — no rebuild.

### Denying an entire tool
Adding a bare-name deny (`Bash`, `WebFetch`, `Bash(*)`, or `"*"` tool-name glob) removes the tool from Claude's context = system-prompt change = invalidation.

**Scoped denies like `Bash(rm *)`, all allow rules, and all ask rules don't change which tools Claude sees** — they're checked when Claude attempts a call, the prefix stays intact.

An `mcp__*` glob also removes those tools but leaves cache intact when matched tools are deferred (the default) — they were never in the cached prefix.

### Compacting the conversation
By design — `/compact` replaces history with a shorter summary that no longer shares the prefix.

**Subtlety**: the summarization call itself shares your prefix, so it reads the existing cache. The slow part is *generating* the summary, not the cache miss. The post-compaction turn rebuilds cache only for the much shorter summary.

### Upgrading Claude Code
A new version typically updates the system prompt or tool definitions. Auto-update downloads in the background but applies on the next launch — never mid-session.

> **Worst-case turn**: resuming (`--resume`) a long session after an upgrade reprocesses the entire history under the new system prompt. The longer the resumed conversation, the more expensive the first turn back.

Set `DISABLE_AUTOUPDATER=1` to control when upgrades apply.

## Actions that keep the cache

These either append to the conversation or don't touch the request.

### Editing files in your repo
File contents enter context only when Claude reads them. **Editing a file Claude previously read doesn't retroactively change the earlier read** — Claude Code appends a `<system-reminder>` noting the change and Claude re-reads if needed.

### Editing `CLAUDE.md` mid-session
Cache stays — **but the edit also doesn't apply.** Project-root + user `CLAUDE.md` are loaded once at session start. The new content loads on the next `/clear`, `/compact`, or restart.

Nested `CLAUDE.md` files in subdirs and rules with `paths:` frontmatter behave differently because they load lazily:
- Edit *before* first load → takes effect on first load.
- Edit *after* it loads → mid-session edit doesn't retroactively change. Same as files.

> **Practical**: if you change `CLAUDE.md` for an in-progress session to take effect, you need `/clear` or restart. Mid-session memory tweaks are usually a sign the session has run its course.

### Changing output style
Same as `CLAUDE.md`: cache safe, but the change doesn't apply until next `/clear` or restart. It's part of the system prompt, baked in at session start.

### Changing permission mode
Cache safe. Mode switches don't touch the system prompt or tool definitions.

**Exception**: plan mode with `opusplan` swaps models = invalidation.

### Invoking skills and commands
Injected as user messages at the point of invocation — nothing earlier changes.

### `/recap`
Appends summary as command output rather than replacing history. Cached prefix stays intact.

### `/rewind`
Truncates the conversation back to an earlier turn. The remaining history is what the cache was built from at that point → next request hits an **earlier cache entry**. Reads on every turn since kept the entry warm even if the original turn was longer ago than the TTL.

> **Bigger-picture tip**: if you've gone down a path you want to abandon, `/rewind` is faster than `/compact` because it truncates back to a prefix that's already cached, rather than building a new one.

### Spawning a subagent
Parent cache **unaffected** — the subagent call + result append to your conversation, prefix intact.

Subagent builds its own cache from scratch (own system prompt, own tools). Subagents use the **5-minute TTL** even on a subscription, since the automatic 1-hour TTL is for the main conversation.

A **fork** (`/fork`) is different: it inherits the parent's system prompt, tools, and conversation history exactly, so its first request **reads the parent's cache**.

## Cache lifetime

Cache entries expire after a period of inactivity. Each cache hit resets the timer. Two TTLs available:

| TTL | Default for |
| --- | --- |
| **5 minutes** | API key, Bedrock, Vertex AI, Foundry, Claude Platform on AWS, **subscription using usage credits** |
| **1 hour** | Claude subscription within plan limits (auto-applied — costs nothing extra since usage is included) |

**Override via env vars:**

- `ENABLE_PROMPT_CACHING_1H=1` — opt into 1-hour on API/cloud providers (bills cache writes at a higher rate).
- `FORCE_PROMPT_CACHING_5M=1` — force 5-minute regardless of auth (useful for debugging, or to override a managed-settings 1-hour).

On **Bedrock**, cacheable prefix length and 1-hour TTL availability vary by model. If cache token counts stay at zero, check the Bedrock docs for your model + region.

## Cache scope (the per-machine, per-directory gotcha)

The cache is **effectively scoped to one machine and one directory**. The system prompt embeds the working directory, platform, shell, OS version, and auto-memory paths — so two sessions in **different directories build different prefixes and miss each other's cache**.

That includes:

- **Worktrees of the same repo** — each has its own CWD = its own cache.
- **The macOS host vs the Windows VM** — completely separate caches by definition (different platform + OS version in the system prompt). When you switch from Mac to Windows mid-task, the Windows session rebuilds from zero.
- **Sequential sessions in the same directory** share the prefix **only when the git status snapshot at startup matches** (system prompt also captures branch + recent commits). A new commit between two sessions in the same dir → cache miss on session start.

**Parallel sessions in the same directory** build matching prefixes and read each other's cache.

Under the hood the API cache is broader (isolated per organization, sometimes per workspace) — for Agent SDK callers running fleets, see `agent-sdk/modifying-system-prompts#improve-prompt-caching-across-users-and-machines` to suppress per-machine sections of the system prompt.

## Measuring cache performance

Every API response returns two token counts:

| Field | Meaning |
| --- | --- |
| `cache_creation_input_tokens` | Tokens written to cache this turn — billed at cache write rate. |
| `cache_read_input_tokens` | Tokens served from cache — billed at ~10% of standard input rate. |

**High read-to-creation ratio = caching is working.** If creation stays high turn after turn, something in your prefix is changing — check the invalidation list.

Two ways to watch:

- **Statusline script** reading the `current_usage` object — live view per session. See `statusline` docs.
- **OpenTelemetry exporter** reports cache read/creation per user + session for org-wide visibility.

## Disabling the cache (debug only)

For normal use, leave caching on. To disable:

| Env var | Effect |
| --- | --- |
| `DISABLE_PROMPT_CACHING=1` | All models |
| `DISABLE_PROMPT_CACHING_HAIKU=1` | Haiku only |
| `DISABLE_PROMPT_CACHING_SONNET=1` | Sonnet only |
| `DISABLE_PROMPT_CACHING_OPUS=1` | Opus only |
| `DISABLE_PROMPT_CACHING_FABLE=1` | Fable only |

Set in `settings.json` → `env` block, or in managed settings for org-wide.

## The playbook

| Habit | Why |
| --- | --- |
| **Pick model + effort at session start.** | Mid-session `/model` or `/effort` = full re-read. |
| **Don't bounce in and out of plan mode with `opusplan`.** | Each toggle = model switch = invalidation. Use a single mode when iterating. |
| **Enable fast mode at session start, not mid-task.** | First fast-mode turn costs once. After that, toggles are free (v2.1.86+). |
| **Edit `CLAUDE.md` between sessions, not during.** | Mid-session edits don't apply anyway. `/clear` or restart is the real cost. |
| **Use `/rewind` to abandon a bad path, not `/compact`.** | Rewind hits an existing cache entry. Compact builds a new prefix. |
| **`/compact` at natural task breaks**, not mid-task. | You control when the cost happens. |
| **Delegate research to subagents.** | Parent cache unaffected. Subagent rebuilds its own cache from zero — fine, since its work was going to cost regardless. |
| **`/reload-plugins --force` only when you're ready to pay.** | The reload is when a full re-read happens, not at `/plugin enable`. |
| **Watch the cache numbers when something feels slow.** | Statusline with `current_usage` is the easiest live view. |
| **Don't resume long sessions immediately after a Claude Code upgrade.** | Worst-case turn — entire history reprocessed under new system prompt. |

## Cross-OS notes

- **Each machine has its own cache.** Switching from the Mac to the Windows VM mid-task means rebuilding from zero on the Windows side. Not a per-OS misconfiguration; it's structural.
- **Worktrees split cache.** If you use `--worktree` for parallel sessions on one machine, each worktree builds its own prefix.
- **Auto-update timing**: on Windows (WinGet, manual updater) the upgrade-then-resume cost only hits when you actually run `winget upgrade Anthropic.ClaudeCode`. On Mac (Homebrew, manual) same. On the native installer (auto-updating), the cost arrives whenever you next launch after a background download. Set `DISABLE_AUTOUPDATER=1` if you want to control timing.

## Practical applications for this repo

- For the playbook itself we're rarely hammering the cache — short docs sessions, mostly. Worth knowing when we start drafting longer interactive workflows.
- When we add hooks (`claude/config/hooks.md` later), the cache implications of `disableAllHooks` vs adding a bare-tool deny rule will matter for the writeup.
- The Mac → Windows VM cache-from-zero is worth noting in `claude/setup/` whenever we draft the cross-OS install steps — set expectations that the first session on a fresh OS is the slow one.
