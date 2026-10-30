---
verified_against: 2.1.308
verified_on: 2026-10-27
sources:
  - https://code.claude.com/docs/en/setup
---

# Install + update Claude Code

Install once, then control *when* the tool changes under you. That second part is the reason this doc exists — everything else is a one-liner.

Written for macOS. Other platforms: [Advanced setup](https://code.claude.com/docs/en/setup).

## Install

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

The channel you pick at install time becomes your auto-update default:

```bash
curl -fsSL https://claude.ai/install.sh | bash -s stable   # or a version: -s 2.1.89
```

Verify:

```bash
claude --version
claude doctor          # health check; also reports the last update attempt
```

### Other install paths

| Path | Auto-updates? |
| --- | --- |
| **Native installer** (above, recommended) | Yes, in background |
| Homebrew | No — `brew upgrade`. Cask name picks the channel (`claude-code` = stable, `claude-code@latest` = latest), and `autoUpdatesChannel` is ignored. |
| npm | No — installs the same native binary. If the npm global dir isn't writable, auto-update fails silently. |
| Desktop app | Self-managed |

`CLAUDE_CODE_PACKAGE_MANAGER_AUTO_UPDATE=1` lets Claude Code run the `brew upgrade` for you in the background.

Requirements: macOS 13.0+, 4 GB RAM, x64 or ARM64, internet access.

## Controlling updates

Native installs check on startup and periodically while running. Updates download in the background and apply **on next start** — never mid-session. That means an upgrade shows up as one uncached first turn after a restart, not a surprise mid-task.

### Channels

| Channel | Behaviour |
| --- | --- |
| `latest` (default) | New releases as they ship |
| `stable` | ~1 week behind; skips releases found to regress |

```json
{ "autoUpdatesChannel": "stable" }
```

`stable` is the lower-friction default for work you can't afford to have wobble.

### Pinning and disabling

| Key / var | Effect |
| --- | --- |
| `minimumVersion` | Floor — updates refuse to go below it. Any settings scope. |
| `requiredMinimumVersion` / `requiredMaximumVersion` | Refuse to *start* outside the range. Managed scope only. |
| `DISABLE_AUTOUPDATER=1` | Stops the background check. `claude update` still works. |
| `DISABLE_UPDATES=1` | Blocks every update path, manual included. |

Manual update any time: `claude update` (works even with `DISABLE_AUTOUPDATER=1`).

Switching `latest` → `stable` while on a newer build prompts you to either stay put (pins `minimumVersion` to current) or allow the downgrade.

## Elsewhere in the docs

Not reproduced here — reach for these only when you need them:

- [Linux package managers](https://code.claude.com/docs/en/setup) (apt/dnf/apk, signed repos) and Windows/WSL install.
- [Binary integrity verification](https://code.claude.com/docs/en/setup) — GPG signature and checksum steps for the release manifest.
- [Network configuration](https://code.claude.com/docs/en/network-config) — proxies, firewalls, required endpoints.
- [Troubleshoot installation and login](https://code.claude.com/docs/en/troubleshoot-install) — `command not found`, failed updates, ripgrep issues.
- Uninstall: `rm -f ~/.local/bin/claude && rm -rf ~/.local/share/claude`. Config lives in `~/.claude/` and `./.claude/`.

## See also

- [`authentication.md`](authentication.md) — signing in, credential storage, API-key precedence.
- [`vs-code.md`](vs-code.md) — the extension, and why you still need this CLI install.
- [`../../shared/project-setup/new-machine.md`](../../shared/project-setup/new-machine.md) — the once-per-machine sequence this fits into.
