# Authentication

How Claude Code knows who you are — login methods, credential storage per OS, precedence when multiple credentials are present, long-lived tokens for CI. Distilled from `https://code.claude.com/docs/en/authentication`.

## Mental model

Six possible authentication paths. **Free Claude.ai plan does NOT include Claude Code access** — you need one of these:

| Path | Best for |
| --- | --- |
| Claude Pro / Max subscription | Solo dev |
| Claude for Teams | Smaller team with self-service billing + admin tools |
| Claude for Enterprise | Larger org needing SSO, domain capture, RBAC, compliance, managed policy |
| Claude Console | API-based billing, admin invites users with roles |
| Amazon Bedrock / Google Vertex AI / Microsoft Foundry | Org already on that cloud — env-var-driven, no browser |
| Long-lived OAuth token | CI / scripts / containers |

## First login

```bash
claude
```

First launch opens a browser for OAuth. Three fallbacks if the browser doesn't open automatically:

| Symptom | What to do |
| --- | --- |
| Browser doesn't open at all | Press `c` to copy the login URL to clipboard. Paste into browser manually. |
| Browser shows a login code (not redirected back) | Paste the code into the terminal at `Paste code here if prompted`. |
| You're in WSL2 / SSH / container | Expect the code-paste flow above — browser can't reach Claude Code's local callback. |

`/logout` to re-authenticate.

## Account types — quick setup

### Pro / Max (solo)

Subscribe at `claude.com/pricing`, then `claude` and follow the browser prompts.

### Claude for Teams / Enterprise

1. Admin subscribes (Teams) or contacts sales (Enterprise).
2. Admin invites members from the admin dashboard.
3. Members install Claude Code + log in with their Claude.ai account.

**Enterprise-only features**: SSO, domain capture, RBAC, compliance API, **managed policy settings** (the `managed-settings.json` mechanism covered in `../config/settings.md`).

### Claude Console (API-based billing)

Admin path:
1. Use existing Console account or create one.
2. Add users: bulk-invite via Settings → Members → Invite, **or** set up SSO.
3. Assign one of two roles:
   - **Claude Code** role: can only create Claude Code API keys.
   - **Developer** role: can create any kind of API key.

User path: accept invite → install Claude Code → log in with Console credentials.

### Cloud providers (Bedrock / Vertex / Foundry)

No browser login — env-var-driven.

1. Follow the provider's setup doc.
2. Set the required env vars + cloud credentials.
3. Install Claude Code and run `claude`.

The presence of `CLAUDE_CODE_USE_BEDROCK=1`, `_USE_VERTEX=1`, or `_USE_FOUNDRY=1` is what flips into cloud auth (see precedence below).

## Credential storage (cross-OS)

| OS | Location | Protection |
| --- | --- | --- |
| **macOS** | Encrypted Keychain | OS-managed |
| **Linux** | `~/.claude/.credentials.json` | File mode `0600` |
| **Windows** | `%USERPROFILE%\.claude\.credentials.json` | Inherits user-profile ACLs (your account only by default) |

`CLAUDE_CONFIG_DIR` env var overrides the location on Linux + Windows (not macOS — Keychain stays).

Managed via `/login` and `/logout`. **Don't edit `.credentials.json` by hand.** For custom API endpoints use `ANTHROPIC_BASE_URL` (e.g. LLM gateways).

## Authentication precedence

When multiple credentials are present, Claude Code picks one in this order — **highest first**:

| Order | Credential | When used |
| --- | --- | --- |
| 1 | Cloud provider creds | `CLAUDE_CODE_USE_BEDROCK` / `_USE_VERTEX` / `_USE_FOUNDRY` is set |
| 2 | `ANTHROPIC_AUTH_TOKEN` | Sent as `Authorization: Bearer`. For LLM gateways using bearer tokens. |
| 3 | `ANTHROPIC_API_KEY` | Sent as `X-Api-Key`. Direct Anthropic API access with a key from `platform.claude.com`. |
| 4 | `apiKeyHelper` script | Dynamic / rotating creds — short-lived tokens from a vault, etc. |
| 5 | `CLAUDE_CODE_OAUTH_TOKEN` | Long-lived OAuth token from `claude setup-token`. For CI/scripts. |
| 6 | Subscription OAuth from `/login` | Default for Pro/Max/Team/Enterprise. |

### The Pro + API key gotcha

If you have an active Claude subscription **and** `ANTHROPIC_API_KEY` is set in your environment, **the API key wins** once approved. Causes authentication failures if the API key belongs to a disabled or expired org.

Recovery:
```bash
unset ANTHROPIC_API_KEY
```
Then `/status` to confirm which method is active.

In interactive mode, you get prompted once to approve or decline the key, and your choice is remembered. Change later via "Use custom API key" toggle in `/config`. In `-p` (non-interactive), the key is **always** used when present (no prompt).

### Where env-var auth applies

`apiKeyHelper`, `ANTHROPIC_API_KEY`, and `ANTHROPIC_AUTH_TOKEN` apply to:
- The CLI
- VS Code extension
- Agent SDK
- GitHub Actions

They do **NOT** apply to:
- Claude Desktop (uses OAuth, unless using an org-distributed third-party inference config)
- Cloud sessions (Claude Code on the Web) — these always use subscription credentials. Env vars in the sandbox environment don't override.

## `apiKeyHelper`

Setting in `settings.json` — points at a shell script that returns an API key. For dynamic or rotating credentials.

```json
{
  "apiKeyHelper": "/bin/generate_temp_api_key.sh"
}
```

Default refresh: every **5 minutes** or on HTTP 401 response. Override with `CLAUDE_CODE_API_KEY_HELPER_TTL_MS`.

**Slow-helper warning**: if the script takes >10s, Claude Code shows an elapsed-time notice in the prompt bar. Optimize the script if you see this regularly.

The returned key is sent the same way as `ANTHROPIC_API_KEY` (precedence 4, but kicks in only when 1–3 are absent).

## Long-lived OAuth token (for CI)

For CI pipelines, scripts, or containers where browser login isn't available:

```bash
claude setup-token
```

Walks you through OAuth + prints a **one-year token**. The command does NOT save it — you copy and store it:

```bash
export CLAUDE_CODE_OAUTH_TOKEN=your-token
```

Constraints:
- Requires Pro / Max / Team / Enterprise plan.
- Authenticates with your Claude subscription.
- **Inference only** — cannot establish Remote Control sessions.
- **Bare mode (`--bare`) doesn't read this.** For bare mode, use `ANTHROPIC_API_KEY` or `apiKeyHelper`.

## Related settings (managed policy)

Covered in `../config/settings.md`. Worth knowing they exist:

| Setting | Effect |
| --- | --- |
| `forceLoginMethod` | `claudeai` restricts to Claude.ai logins; `console` restricts to Console. Blocks env-var auth (API key / auth token / apiKeyHelper) at startup. Cloud-provider sessions exempt. |
| `forceLoginOrgUUID` | Require login to belong to a specific Anthropic org (single UUID or array). Blocks env-var auth too. |

These are managed-policy-only — useful when an org admin needs hard guarantees.

## Verifying what's active

```
/status
```

Status tab shows the active authentication method. Setting Sources line shows which settings layers loaded (covered in `../config/settings.md`).

For env-var-driven configurations, also check:
```bash
echo $ANTHROPIC_API_KEY      # should be empty if you want subscription
echo $CLAUDE_CODE_USE_BEDROCK # 1 means Bedrock path active
```

## Cross-OS notes

- **macOS** is the easiest path — Keychain handles credential storage without any config.
- **Windows** native: same `%USERPROFILE%\.claude\.credentials.json` ACL story. Default file permissions are fine.
- **WSL2** is the most common source of "browser code paste" friction — expect it.
- **`CLAUDE_CONFIG_DIR`** can move the credentials file on Linux + Windows (not macOS) — useful for setups where home directory isn't writable (some containers, restricted CI).

## Practical applications for this repo

- **For solo dev (this user)**: Pro or Max subscription, `/login` once per machine. Done.
- **For the Windows VM**: same login on first run. Credentials stored in `%USERPROFILE%\.claude\.credentials.json`, isolated per machine — not synced with the Mac host. **Don't try to copy the file across** — Keychain on Mac doesn't map to a flat file anyway.
- **For client engagements**: if the client is on Bedrock/Vertex/Foundry, set the right `CLAUDE_CODE_USE_*` env var **before** launching `claude`. If on Console, the client's admin needs to invite your Claude.ai email with the right role.
- **For any CI work that may come up**: `claude setup-token` once, store `CLAUDE_CODE_OAUTH_TOKEN` as a CI secret. The token is good for a year — calendar a renewal.
- **The Pro + `ANTHROPIC_API_KEY` gotcha** is worth catching early. If `/status` shows the wrong method, check env first.
